---
title: "[翻译]使用Architecture Components常见的五个错误"
date: 2018-11-13 17:33:45
tags: 翻译 android
---

原文地址:[5 common mistakes when using Architecture Components](https://proandroiddev.com/5-common-mistakes-when-using-architecture-components-403e9899f4cb)

文章包含以下内容:

- Fragments 中 LiveData Observers 泄漏
- 每次屏幕旋转都会重新加载数据
- ViewModels 泄漏
- 将 LiveData 公开为可视的视图
- 每次配置更改之后创建 ViewModels 依赖

### Fragments 中 LiveData Observers 泄漏

Fragments 的生命周期是比较棘手的，一个 fragment 从 detached 到 attached 的过程中并不总会调用 onDestroy 方法,例如，当配置更改时,fragments 就不会被销毁，当配置更改时，只有 fragments 中的 views 会被销毁，而 fragment 会继续存活,所以 DESTROYED 状态就走不到。

这意味着，我们在 onCreateView 或 onActivityCreated 中开始观察 LiveData 时，就会将 fragment 作为 LifecycleOwner:

```java
class BooksFragment: Fragment() {

    private lateinit var viewModel: BooksViewModel

    override fun onCreateView(inflater: LayoutInflater, container: ViewGroup?, savedInstanceState: Bundle?): View? {
        return inflater.inflate(R.layout.fragment_books, container)
    }

    override fun onActivityCreated(savedInstanceState: Bundle?) {
        super.onActivityCreated(savedInstanceState)
        viewModel = ViewModelProviders.of(this).get(BooksViewModel::class.java)

        viewModel.liveData.observe(this, Observer { updateViews(it) })  // Risky: Passing Fragment as LifecycleOwner
    }

    ...
}
```

每次 fragment 重新 attached 时会创建一个新的 Observer 实例，但是之前的 Observer 并没有移除，因为 LifecycleOwner 没有走到**DESTROYED**状态，这会导致越来越多的相同观察者同时处于活动状态，并且 onChanged 中的代码被多次执行。

这个问题可以在[这里](https://medium.com/@BladeCoder/architecture-components-pitfalls-part-1-9300dd969808)找到更多的解释

最终解决方案就是使用 fragments 中 view 的生命周期开来代替 fragments 的生命周期，**getViewLifeCycleOwner()**和**getViewLifeCycleOwnerLiveData()**在 support library 28.0.0 和 AndroidX 1.0.0 中被提供,这样每次 fragment 中 views 被销毁的时候，LiveData 都会移除掉 observers。

```java
class BooksFragment : Fragment() {

    ...

    override fun onActivityCreated(savedInstanceState: Bundle?) {
        super.onActivityCreated(savedInstanceState)
        viewModel = ViewModelProviders.of(this).get(BooksViewModel::class.java)

        viewModel.liveData.observe(viewLifecycleOwner, Observer { updateViews(it) })    // Usually what we want: Passing Fragment's view as LifecycleOwner
    }

    ...
}
```

### 每次屏幕旋转都重新加载数据

通常情况下初始化和设置逻辑我们都会放在 Activity 中的 onCreate()方法中，通常我们也会在 onCreate 中触发加载数据的逻辑，但是会产生的问题就是当屏幕旋转的时候会重新加载数据

```java
class ProductViewModel(
    private val repository: ProductRepository
) : ViewModel() {

    private val productDetails = MutableLiveData<Resource<ProductDetails>>()
    private val specialOffers = MutableLiveData<Resource<SpecialOffers>>()

    fun getProductsDetails(): LiveData<Resource<ProductDetails>> {
        repository.getProductDetails()  // Loading ProductDetails from network/database
        ...                             // Getting ProductDetails from repository and updating productDetails LiveData
        return productDetails
    }

    fun loadSpecialOffers() {
        repository.getSpecialOffers()   // Loading SpecialOffers from network/database
        ...                             // Getting SpecialOffers from repository and updating specialOffers LiveData
    }
}

class ProductActivity : AppCompatActivity() {

    lateinit var productViewModelFactory: ProductViewModelFactory

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val viewModel = ViewModelProviders.of(this, productViewModelFactory).get(ProductViewModel::class.java)

        viewModel.getProductsDetails().observe(this, Observer { /*...*/ })  // (probable) Reloading product details after every rotation
        viewModel.loadSpecialOffers()                                       // (probable) Reloading special offers after every rotation
    }
}
```

解决方案：

1. 使用 AbsentLiveData，只有在数据没有被设置的时候才去加载
2. 在真正需要的时候才开始加载数据，比如在 OnClickListener 中
3. 在 ViewModels 的构造函数中加载并暴露 getters。

```java
class ProductViewModel(
    private val repository: ProductRepository
) : ViewModel() {

    private val productDetails = MutableLiveData<Resource<ProductDetails>>()
    private val specialOffers = MutableLiveData<Resource<SpecialOffers>>()

    init {
        loadProductsDetails()           // ViewModel is created only once during Activity/Fragment lifetime
    }

    private fun loadProductsDetails() { // private, just utility method to be invoked in constructor
        repository.getProductDetails()  // Loading ProductDetails from network/database
        ...                             // Getting ProductDetails from repository and updating productDetails LiveData
    }

    fun loadSpecialOffers() {           // public, intended to be invoked by other classes when needed
        repository.getSpecialOffers()   // Loading SpecialOffers from network/database
        ...                             // Getting SpecialOffers from repository and updating _specialOffers LiveData
    }

    fun getProductDetails(): LiveData<Resource<ProductDetails>> {   // Simple getter
        return productDetails
    }

    fun getSpecialOffers(): LiveData<Resource<SpecialOffers>> {     // Simple getter
        return specialOffers
    }
}

class ProductActivity : AppCompatActivity() {

    lateinit var productViewModelFactory: ProductViewModelFactory

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val viewModel = ViewModelProviders.of(this, productViewModelFactory).get(ProductViewModel::class.java)

        viewModel.getProductDetails().observe(this, Observer { /*...*/ })    // Just setting observer
        viewModel.getSpecialOffers().observe(this, Observer { /*...*/ })     // Just setting observer

        button_offers.setOnClickListener { viewModel.loadSpecialOffers() }
    }
}
```

###  泄漏 ViewModels

我们不应该将 View 的引用传递给 ViewModel，但是我们也应该谨慎的将对 ViewModels 的引用传递给其他类，当 Activity 已经 finished，ViewModel 也应该进行垃圾回收。

示例中是传递一个监听器到 Repository，他是 Singleton 范围的，并且之后不会清楚监听器。

```java
@Singleton
class LocationRepository() {

    private var listener: ((Location) -> Unit)? = null

    fun setOnLocationChangedListener(listener: (Location) -> Unit) {
        this.listener = listener
    }

    private fun onLocationUpdated(location: Location) {
        listener?.invoke(location)
    }
}


class MapViewModel: AutoClearViewModel() {

    private val liveData = MutableLiveData<LocationRepository.Location>()
    private val repository = LocationRepository()

    init {
        repository.setOnLocationChangedListener {   // Risky: Passing listener (which holds reference to the MapViewModel)
            liveData.value = it                     // to singleton scoped LocationRepository
        }
    }
}
```

解决方法就是在**onCleared()**方法中移除掉监听器，在 Repository 中将 listener 存储为 **WeakReference**，使用 LiveData 在 Repository 和 ViewModel 之间进行通信。

```java
@Singleton
class LocationRepository() {

    private var listener: ((Location) -> Unit)? = null

    fun setOnLocationChangedListener(listener: (Location) -> Unit) {
        this.listener = listener
    }

    fun removeOnLocationChangedListener() {
        this.listener = null
    }

    private fun onLocationUpdated(location: Location) {
        listener?.invoke(location)
    }
}


class MapViewModel: AutoClearViewModel() {

    private val liveData = MutableLiveData<LocationRepository.Location>()
    private val repository = LocationRepository()

    init {
        repository.setOnLocationChangedListener {   // Risky: Passing listener (which holds reference to the MapViewModel)
            liveData.value = it                     // to singleton scoped LocationRepository
        }
    }

    override onCleared() {                            // GOOD: Listener instance from above and MapViewModel
        repository.removeOnLocationChangedListener()  //       can now be garbage collected
    }
}
```

### 将 LiveData 公开为可视的视图

视图应该只观察 LiveData，yincident 我们应该封装对 MutableLiveData 的访问。

```java
class CatalogueViewModel : ViewModel() {

    // BAD: Exposing mutable LiveData
    val products = MutableLiveData<Products>()


    // GOOD: Encapsulate access to mutable LiveData through getter
    private val promotions = MutableLiveData<Promotions>()

    fun getPromotions(): LiveData<Promotions> = promotions


    // GOOD: Encapsulate access to mutable LiveData using backing property
    private val _offers = MutableLiveData<Offers>()
    val offers: LiveData<Offers> = _offers


    fun loadData(){
        products.value = loadProducts()     // Other classes can also set products value
        promotions.value = loadPromotions() // Only CatalogueViewModel can set promotions value
        _offers.value = loadOffers()        // Only CatalogueViewModel can set offers value
    }
}
```

### 每次配置更改之后创建 ViewModel 依赖

ViewModel 可以在配置更改之类的操作中存活，所以每次配置更改时创建它的依赖关系是多余的，比如下面的代码：

```java
class MoviesViewModel(
    private val repository: MoviesRepository,
    private val stringProvider: StringProvider,
    private val authorisationService: AuthorisationService
) : ViewModel() {

    ...
}


class MoviesViewModelFactory(   // We need to create instances of below dependencies to create instance of MoviesViewModelFactory
    private val repository: MoviesRepository,
    private val stringProvider: StringProvider,
    private val authorisationService: AuthorisationService
) : ViewModelProvider.Factory {

    override fun <T : ViewModel> create(modelClass: Class<T>): T {  // but this method is called by ViewModelProvider only if ViewModel wasn't already created
        return MoviesViewModel(repository, stringProvider, authorisationService) as T
    }
}


class MoviesActivity : AppCompatActivity() {

    @Inject
    lateinit var viewModelFactory: MoviesViewModelFactory

    private lateinit var viewModel: MoviesViewModel

    override fun onCreate(savedInstanceState: Bundle?) {    // Called each time Activity is recreated
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_movies)
        injectDependencies() // Creating new instance of MoviesViewModelFactory
        viewModel = ViewModelProviders.of(this, viewModelFactory).get(MoviesViewModel::class.java)
    }

    ...
}
```

每次配置更改时都需要创建一个新的 ViewModelFactory 实例，因此不必要创建其所有依赖项的新实例。

解决方案就是延迟创建依赖直到 create 方法调用时， 因为他在 Activity/Fragment 生命周期内只调用一次。我们可以使用延迟初始化来实现这一点：

```java
class MoviesViewModel(
    private val repository: MoviesRepository,
    private val stringProvider: StringProvider,
    private val authorisationService: AuthorisationService
) : ViewModel() {

    ...
}


class MoviesViewModelFactory(
    private val repository: Provider<MoviesRepository>,             // Passing Providers here
    private val stringProvider: Provider<StringProvider>,           // instead of passing directly dependencies
    private val authorisationService: Provider<AuthorisationService>
) : ViewModelProvider.Factory {

    override fun <T : ViewModel> create(modelClass: Class<T>): T {  // This method is called by ViewModelProvider only if ViewModel wasn't already created
        return MoviesViewModel(repository.get(),
                               stringProvider.get(),                // Deferred creating dependencies only if new insance of ViewModel is needed
                               authorisationService.get()
                              ) as T
    }
}


class MoviesActivity : AppCompatActivity() {

    @Inject
    lateinit var viewModelFactory: MoviesViewModelFactory

    private lateinit var viewModel: MoviesViewModel

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_movies)

        injectDependencies() // Creating new instance of MoviesViewModelFactory
        viewModel = ViewModelProviders.of(this, viewModelFactory).get(MoviesViewModel::class.java)
    }

    ...
}
```
