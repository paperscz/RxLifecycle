## RxLifecycle-Retrofit
### 首先说明这个方式的实现原理是:通过自定义CallAdapterFactory,将Observable与activity绑定.每一个Activity绑定一个retrofit客户端,在每次创建service接口的时候,都要重新初始化Retrofit客户端,禁止复用.
### 这个模块只有两个类:
 - HttpHelper : 示例工具类,创建和界面绑定的service接口.可以直使用.
 - RxJavaLifecycleCallAdapterFactory : RxJavaCallAdapterFactory扩展类,将接口请求绑定到对应Activity的onDestroy.
### gradle依赖不变:
``` 
		//RxJava1
		compile 'com.dhh:rxlifecycle:1.5'
		//RxJava2
		compile 'com.dhh:rxlifecycle2:1.5'

```
## How to use

### 一. 如果你自己没有自定义RxJavaCallAdapterFactory,直接忽略这条操作.若你已经自定义了一个XXXRxJavaCallAdapterFactory(不是Retrofit自带的那个),又想使用本库提供的RxJavaLifecycleCallAdapterFactory,那么你需要这么做:
###在你项目的Application的onCreate方法中进行初始化:
```
		//RxJava1
        RxJavaLifecycleCallAdapterFactory.injectCallAdapterFactory(yourFactory);
		//RxJava2
        RxJava2LifecycleCallAdapterFactory.injectCallAdapterFactory(yourFactory);


```
## ★★★ 如果在你的项目里没有自定义 XXXRxJavaCallAdapterFactory,请忽略第一步的配置.
### 二. 在使用retrofit将接口实例化的时候,切记一定要一同初始化一个新的Retrofit客户端,在addCallAdapterFactory的时候用项目里提供的RxJavaLifecycleCallAdapterFactory.createWithLifecycleManager(lifecycleManager),你可以在自己的项目里封装一个HttpHelper类,单例模式,将okhttpClient,以及baseUrl等等相关配置的东西都预先处理好,库里提供了一个HttpHelper类,你可以参考这个类,进行扩展,代码如下:
```

		public class HttpHelper {
		    private static HttpHelper instance;
		    public String baseUrl;
		    private OkHttpClient client;
		    private CallAdapter.Factory callAdapterFactory;
		    private Converter.Factory converterFactory;
		
		    public HttpHelper() {
		        client = new OkHttpClient.Builder()
		                //other config ...
		
		                .build();
		        callAdapterFactory = RxJavaLifecycleCallAdapterFactory.create();
		    }
		
		    public static HttpHelper getInstance() {
		        if (instance == null) {
		            synchronized (HttpHelper.class) {
		                if (instance == null) {
		                    instance = new HttpHelper();
		                }
		            }
		        }
		        return instance;
		    }
		
		    /**
		     * 不带生命周期自动绑定的
		     *
		     * @param service
		     * @param baseUrl
		     * @param <T>
		     * @return
		     */
		    public <T> T create(Class<T> service, String baseUrl) {
		        Retrofit retrofit = new Retrofit.Builder()
		                .baseUrl(baseUrl)
		                .addCallAdapterFactory(callAdapterFactory)
		                .addConverterFactory(checkNotNull(converterFactory,"converterFactory == null"))
		                .client(client)
		                .build();
		        return retrofit.create(service);
		    }
		
		    /**
		     * 不带生命周期自动绑定的
		     *
		     * @param service
		     * @param <T>
		     * @return
		     */
		    public <T> T create(Class<T> service) {
		        return create(service, baseUrl);
		    }
		
		    /**
		     * 带生命周期自动绑定的
		     *
		     * @param service
		     * @param baseUrl
		     * @param lifecycleManager
		     * @param <T>
		     * @return
		     */
		    public <T> T createWithLifecycleManager(Class<T> service, String baseUrl, LifecycleManager lifecycleManager) {
		        Retrofit retrofit = new Retrofit.Builder()
		                .baseUrl(baseUrl)
		                .addCallAdapterFactory(RxJavaLifecycleCallAdapterFactory.createWithLifecycleManager(lifecycleManager))
		                .addConverterFactory(checkNotNull(converterFactory,"converterFactory == null"))
		                .client(client)
		                .build();
		        return retrofit.create(service);
		    }
		
		    /**
		     * 带生命周期自动绑定的
		     *
		     * @param service
		     * @param lifecycleManager
		     * @param <T>
		     * @return
		     */
		    public <T> T createWithLifecycleManager(Class<T> service, LifecycleManager lifecycleManager) {
		        return createWithLifecycleManager(service, baseUrl, lifecycleManager);
		    }
		
		    public void setBaseUrl(String baseUrl) {
		        this.baseUrl = checkNotNull(baseUrl,"baseUrl == null");
		    }
		
		    public OkHttpClient getClient() {
		        return client;
		    }
		
		    public void setClient(OkHttpClient client) {
		        this.client = checkNotNull(client,"client == null");
		    }
		
		    public void setConverterFactory(Converter.Factory converterFactory) {
		        this.converterFactory = checkNotNull(converterFactory,"converterFactory == null");
		    }
		    static <T> T checkNotNull(@Nullable T object, String message) {
		        if (object == null) {
		            throw new NullPointerException(message);
		        }
		        return object;
		    }
		}

```
### 其中 createWithLifecycleManager方法是核心方法,将LifecycleManager传入.
### 如果你使用项目提供的HttpHelper,在使用前必须做一些必要的初始化:
```

        HttpHelper.getInstance().setBaseUrl(baseUrl);
        HttpHelper.getInstance().setClient(new OkHttpClient());
        HttpHelper.getInstance().setConverterFactory(GsonConverterFactory.create());

```
## 三. 实例化接口
### 在调用 createWithLifecycleManager 方法的时候,只需要在调用方法的地方能获取到对应Activity的LifecycleManager即可,比如在Activity中实例化接口(准确地说是动态代理出来的):
```

        LifecycleManager lifecycleManager = RxLifecycle.with(this);
        Api api = HttpHelper.getInstance().createWithLifecycleManager(Api.class, lifecycleManager);
        api.get("https://github.com/dhhAndroid/RxLifecycle")
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Action1<ResponseBody>() {
                    @Override
                    public void call(ResponseBody body) {
                        
                    }
                });

```

### 这样的话,这个get请求如果在Activity销毁的时候(onDestroy)还没有结束,就会直接结束掉.通过这个api接口发出的网络请求都会在Activity在onDestroy的时候取消订阅,防止内存泄漏.其次,我再内部直接将线程指定在RxJava的io线程,外部不用在重复写 subscribeOn(Schedulers.io()) 这行代码.至于 observeOn(AndroidSchedulers.mainThread()) 这行代码我没有在内部封装是因为用户可能要对数据做一些转化处理,也有可能比较耗时,所以切换主线程,在数据转换做完后比较好.
### 当然如果你在网络请求之后,又做了比较耗时的操作,比如下载文件和上传文件,为了安全期间,还是可以在调用subscribe之前加入compose(RxLifecycle.with(this).<String>bindOnDestroy()),代码如下:
```

        api.get("https://github.com/dhhAndroid/RxLifecycle")
                .compose(RxLifecycle.with(this).<ResponseBody>bindOnDestroy())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Action1<ResponseBody>() {
                    @Override
                    public void call(ResponseBody body) {
                        
                    }
                });

```
### HttpHelper调用API完全一样,只是RxJavaCallAdapter不一样.
# 详情请查看demo.
# 详情请查看demo.
# 详情请查看demo.
# 从此就可以对 使用RxJava+Retrofit 导致的内存泄漏说 ( ^_^ )/~~拜拜 !