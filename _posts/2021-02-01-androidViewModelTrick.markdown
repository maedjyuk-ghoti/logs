---
layout: post
title: Android View Model Trick
date: 2021-02-01 18:30:45 -0400
categories: programming kotlin
---

# Introduction
Sometimes you want to share a behavior/listener or a value or a stream between a parent activity/fragment and a child fragment or between fragment peers. But you can't provide a constructor for fragments because Android recreates them using an empty constructor after certain lifecycle events such as screen rotation. Android's ViewModel is an answer here.

The crux of the idea is that ViewModels are associated with a LifecycleOwner. Therefore, as long as both users have access to the same LifecycleOwner, they both have access to the same ViewModel. This works if one of the users is the parent and the other is the child, or if both are children of a common parent. Activities and Fragments are LifecycleOwners. This enables lots of what we're about to see.

# Scenarios
## Parent Activity and Child Fragment
Let's contrive a scenario. Please ignore any faulty LiveData code.

```kotlin
// Need a ViewModel to hold the data we want to display
class MainActivityViewModel : ViewModel() {
    private val _count: MutableLiveData<Int> = MutableLiveData<Int>()
    val count: LiveData<Int>
        get() = _count

    init {
        _count.value = 0
    }

    fun increment() {
        _count.value = _count.value + 1
    }
}

// A fragment to share the ViewModel with
class SomeFragment : Fragment() {
    /**
     * "Current" best practice way to create a viewmodel is to use...
     * private val viewModel: MainActivityViewModel by activityViewModels()
     * But this just obfuscates what's happening so we're going to dig
     * into it a little and remove the first layer of wrapping paper
     */
    private val viewModel: MainActivityViewModel by lazy {
        ViewModelProvider(requireActivity()).get(MainActivityViewModel::class.java)
    }

    init {
        viewModel.increment()
    }
}

// This will be the base activity that, for some reason, has a display box that we want to update
// Assume that SomeFragment was added in the XML, I don't want to clutter the code here
class MainActivity : Activity() {
    /**
     * "Current" best practice way to create a viewmodel is to use...
     * private val viewModel: MainActivityViewModel by viewModels()
     * But this just obfuscates what's happening so we're going to dig
     * into it a little and remove the first layer of wrapping paper
     */
    private val viewModel: MainActivityViewModel by lazy {
        ViewModelProvider(this).get(MainActivityViewModel::class.java)
    }

    // Just some view to display data
    private lateinit var displayBox: TextView

    fun onViewCreated(view: View, savedInstanceState: Bundle) {
        displayBox = view.findViewById(r.id.displayBox)
        viewModel.count.observe(this, Observer { value -> displayBox.setText("$value") })
    }
}
```

In the example above, you can see there is a View in the Activity that holds a count of something and a ViewModel. We obtain this ViewModel by providing the Activity (which subclasses LifecycleOwner) and the ViewModel we want to get and we get a completely fresh and new instance of MainActivityViewModel.

In our Fragment we also have a ViewModel. We obtain this by providing the ParentActivity (which subclasses LifecycleOwner) and the ViewModel we want to get... the exact same instance of MainActivityViewModel that the Activity already has.

This is because Android hides things behind the scenes and ensures that there is only ever 1 ViewModel of a type at a time for each LifecycleOwner. Because we're asking for the same ViewModel by the same LifecycleOwner we get the same ViewModel. Notice, that because it's tied to the LifecycleOwner, the Fragment can still have it's own ViewModel.

```kotlin
class SomeFragmentViewModel : ViewModel() {
    ...
}

// A fragment to share the ViewModel with
class SomeFragment : Fragment() {
    private val viewModel: MainActivityViewModel by lazy {
        ViewModelProvider(requireActivity()).get(MainActivityViewModel::class.java)
    }

    /**
     * "Current" best practice way to create a viewmodel is to use...
     * private val anotherViewModel: SomeFragmentViewModel by viewModels()
     * But this just obfuscates what's happening so we're going to dig
     * into it a little and remove the first layer of wrapping paper
     */
    private val anotherViewModel: SomeFragmentViewModel by lazy {
        ViewModelProvider(this).get(SomeFragmentViewModel::class.java)
    }

    ...
}
```

## Parent Fragment and Child Fragment
The case for parent Fragment and child Fragment is identical to parent Activity and child Fragment, except for the following:
* MainActivity : Activity() would be MainFragment : Fragment()
* requireActivity() would be requireParentFragment()

## Fragment Peers
The case for Fragment Peers is identical to Parent Activity/Fragment and child Fragment, except there would be at least one more fragment.

## Factories
Sometimes you want to instantiate your ViewModel with some values, in which case you would have to create a ViewModelFactory so that Android can recreate your ViewModel in case it deletes it. This is the purpose of the factory. It doesn't change much from the previous examples except in the Parent Activity/Fragment. There the ViewModelFactory will have to be created and used. In all child Fragments, the factory is omitted because the ViewModel should already be created.

```kotlin
class MainActivityViewModel(initialCount: Int) : ViewModel() {
    private val _count: MutableLiveData<Int> = MutableLiveData<Int>()
    
    ...

    init {
        _count.value = initialCount
    }

    ...
}

class MainActivityViewModelFactory(private val initialCount: Int) : ViewModelProvider.Factory() {
    override fun <T : ViewModel?> create(modelClass: Class<T>): T {
        return modelClass.getConstructor(Int::class.java).newInstance(initialCount)
    }
}

class MainActivity : Activity() {
    private val viewModel: MainActivityViewModel by lazy {
        ViewModelProvider(this, MainActivityViewModelFactory(0)).get(MainActivityViewModel::class.java)
    }

    ...
}
```

Here we want an initialCount passed into the MainActivityViewModel so it can be chosen in the Activity instead of prescribed by the ViewModel. We create a factory that Android can use to create our ViewModel, otherwise we'd get a runtime error complaining that the ViewModel can't be created because Android doesn't know what to put in the constructor.