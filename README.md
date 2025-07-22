# 📱 Android Interview Q&A – Deep Dive with Practical Insights

I've provided **comprehensive answers** to commonly asked Android interview questions. Each section includes:

- ✅ Practical code examples showing real implementation  
- 🧠 Key differences highlighted clearly  
- 📌 When to use each approach with real-world scenarios  
- 🚀 Modern alternatives where applicable  

---

## 🔍 Quick Summary

### 📲 Screen Rotation  
- `ViewModel` automatically retains data across configuration changes  
- Without ViewModel: requires manual use of `onSaveInstanceState` / `SavedStateHandle`  

### 🔧 Services  
- **Bound**: Tied to component lifecycle (e.g., Activity), ideal for client-server communication  
- **Unbound**: Runs independently in background  

### ⚖️ Service vs Thread  
- Use **Service** for long-running background operations  
- Use **Thread** for short-lived, UI-related tasks  

### 🏗️ MVVM vs MVP  
- **MVVM** supports better data binding and works well with LiveData, ViewModel  
- **MVP** tightly couples view & presenter, harder to test and maintain  

### 🚀 launch vs async  
- `launch`: Fire-and-forget (does not return result)  
- `async`: Used when a result is expected via `await()`  

### 🧵 Coroutines vs Threads  
- **Coroutines** are lightweight, non-blocking  
- **Threads** are heavyweight, block resources  

### ⏱️ delay() vs Thread.sleep()  
- `delay()` suspends coroutine without blocking thread  
- `Thread.sleep()` blocks the entire thread  

### 🧭 SupervisorJob  
- Child coroutine failure doesn't cancel siblings (vs `Job` where failure propagates)  

### 📬 Handler, Looper, HandlerThread  
- Legacy system for message passing and background tasks  
- Replaced in modern apps with **Coroutines**, **WorkManager**, or **Executors**  

---

💡 *Use this as a cheat sheet or prep guide for your Android interviews!*  


## 1. Android Lifecycle & Screen Rotation Handling

### Without ViewModel:
```kotlin
class MainActivity : AppCompatActivity() {
    private var counter = 0
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        
        // Data is lost on rotation - counter resets to 0
        // Need to manually save/restore state
    }
    
    override fun onSaveInstanceState(outState: Bundle) {
        super.onSaveInstanceState(outState)
        outState.putInt("counter", counter)
    }
    
    override fun onRestoreInstanceState(savedInstanceState: Bundle) {
        super.onRestoreInstanceState(savedInstanceState)
        counter = savedInstanceState.getInt("counter", 0)
    }
}
```

### With ViewModel:
```kotlin
class CounterViewModel : ViewModel() {
    private val _counter = MutableLiveData(0)
    val counter: LiveData<Int> = _counter
    
    fun increment() {
        _counter.value = _counter.value!! + 1
    }
}

class MainActivity : AppCompatActivity() {
    private lateinit var viewModel: CounterViewModel
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        
        viewModel = ViewModelProvider(this)[CounterViewModel::class.java]
        // Data survives configuration changes automatically
        // ViewModel is retained during rotation
    }
}
```

### The Flow During Screen Rotation:
```
User rotates screen
       ↓
Activity.onDestroy() called
       ↓
Android saves ViewModelStore in memory
       ↓
Activity recreated
       ↓
Activity.onCreate() called
       ↓
ViewModelProvider(this) retrieves SAME ViewModelStore
       ↓
Same ViewModel instance returned
       ↓
_counter still has value 5!
```

**Key Differences:**
- **Without ViewModel**: Data is lost, manual state saving required
- **With ViewModel**: Data survives rotation automatically, cleaner code

---

## 2. Types of Services

### Bound Service:
```kotlin
class MusicService : Service() {
    private val binder = MusicBinder()
    
    inner class MusicBinder : Binder() {
        fun getService(): MusicService = this@MusicService
    }
    
    override fun onBind(intent: Intent): IBinder = binder
    
    fun playMusic() {
        // Music playback logic
    }
}

// Client binding
class MainActivity : AppCompatActivity() {
    private var musicService: MusicService? = null
    private var isBound = false
    
    private val connection = object : ServiceConnection {
        override fun onServiceConnected(name: ComponentName?, service: IBinder?) {
            val binder = service as MusicService.MusicBinder
            musicService = binder.getService()
            isBound = true
        }
        
        override fun onServiceDisconnected(name: ComponentName?) {
            isBound = false
        }
    }
}
```

### Unbound Service (Started Service):
```kotlin
class DownloadService : Service() {
    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        // Download logic here
        // Runs independently of client
        downloadFile()
        return START_STICKY // Service restarts if killed
    }
    
    override fun onBind(intent: Intent): IBinder? = null
    
    private fun downloadFile() {
        // Long-running download operation
        stopSelf() // Stop service when done
    }
}

// Starting the service
startService(Intent(this, DownloadService::class.java))
```

**Key Differences:**
- **Bound Service**: Lives only while clients are bound, direct communication
- **Unbound Service**: Runs independently, fire-and-forget operations

---

## 3. Service vs Thread - When to Use Each

### Use Service When:
- **Background work** that should continue even when app is not visible
- **Long-running operations** (file downloads, music playback)
- **System-level operations** that need to persist
- **Cross-component communication** needed

```kotlin
// Example: Background file upload
class UploadService : Service() {
    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        // Uploads continue even if user leaves app
        uploadFiles()
        return START_STICKY
    }
}
```

### Use Thread When:
- **UI-related background work** within an activity/fragment
- **Short-term operations** that don't need to persist
- **CPU-intensive tasks** that should pause when app is not active

```kotlin
// Example: Image processing within an activity
class ImageActivity : AppCompatActivity() {
    private fun processImage() {
        Thread {
            // Process image on background thread
            val processedImage = heavyImageProcessing()
            
            runOnUiThread {
                // Update UI with result
                imageView.setImageBitmap(processedImage)
            }
        }.start()
    }
}
```

---

## 4. MVVM Architecture

### MVVM Components:
```kotlin
// Model
data class User(val id: Int, val name: String, val email: String)

// Repository
class UserRepository {
    suspend fun getUsers(): List<User> {
        return apiService.getUsers()
    }
}

// ViewModel
class UserViewModel(private val repository: UserRepository) : ViewModel() {
    private val _users = MutableLiveData<List<User>>()
    val users: LiveData<List<User>> = _users
    
    private val _loading = MutableLiveData<Boolean>()
    val loading: LiveData<Boolean> = _loading
    
    fun loadUsers() {
        viewModelScope.launch {
            _loading.value = true
            try {
                _users.value = repository.getUsers()
            } catch (e: Exception) {
                // Handle error
            } finally {
                _loading.value = false
            }
        }
    }
}

// View (Activity/Fragment)
class UserActivity : AppCompatActivity() {
    private lateinit var viewModel: UserViewModel
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        viewModel.users.observe(this) { users ->
            // Update UI with user list
            adapter.updateUsers(users)
        }
        
        viewModel.loading.observe(this) { isLoading ->
            progressBar.isVisible = isLoading
        }
        
        viewModel.loadUsers()
    }
}
```

### MVVM vs MVP:

| Aspect | MVVM | MVP |
|--------|------|-----|
| **Data Binding** | Two-way binding with LiveData/DataBinding | Manual view updates |
| **View Dependency** | ViewModel doesn't know about View | Presenter holds View reference |
| **Testability** | Easy to test ViewModel independently | Presenter testing requires mock views |
| **Configuration Changes** | ViewModel survives rotation | Presenter needs recreation |
| **Architecture Components** | Built-in support (ViewModel, LiveData) | Manual implementation needed |

**MVVM Example:**
```kotlin
// ViewModel doesn't reference View
class LoginViewModel : ViewModel() {
    val loginResult = MutableLiveData<Boolean>()
    
    fun login(username: String, password: String) {
        // Business logic
        loginResult.value = authenticate(username, password)
    }
}
```

**MVP Example:**
```kotlin
// Presenter holds View reference
class LoginPresenter(private val view: LoginView) {
    fun login(username: String, password: String) {
        val isSuccess = authenticate(username, password)
        if (isSuccess) {
            view.showSuccess()
        } else {
            view.showError()
        }
    }
}
```

---

## 5. launch vs async in Coroutines

### launch:
```kotlin
// Fire-and-forget, returns Job
fun loadData() {
    viewModelScope.launch {
        val user = userRepository.getUser()
        // Update UI
        _userData.value = user
    }
    // Doesn't wait for completion
    // Can't get return value
}

// Multiple launches run concurrently
fun loadMultipleData() {
    viewModelScope.launch {
        launch { loadUsers() }    // Concurrent
        launch { loadPosts() }    // Concurrent  
        launch { loadComments() } // Concurrent
    }
}
```

### async:
```kotlin
// Returns Deferred<T>, can get result
fun loadUserProfile() {
    viewModelScope.launch {
        val userDeferred = async { userRepository.getUser() }
        val postsDeferred = async { postRepository.getPosts() }
        
        // Wait for both and combine results
        val user = userDeferred.await()
        val posts = postsDeferred.await()
        
        _userProfile.value = UserProfile(user, posts)
    }
}

// Sequential vs Concurrent with async
fun compareExecution() {
    viewModelScope.launch {
        // Sequential (slower)
        val user1 = getUser(1)
        val user2 = getUser(2)
        
        // Concurrent (faster)
        val user1Deferred = async { getUser(1) }
        val user2Deferred = async { getUser(2) }
        val user1 = user1Deferred.await()
        val user2 = user2Deferred.await()
    }
}
```

**Key Differences:**
- **launch**: Fire-and-forget, returns Job, no return value
- **async**: Returns Deferred<T>, can await() for result, used for concurrent operations that need results

---

## 6. Coroutines vs Threads

### Coroutines:
```kotlin
// Lightweight, cooperative concurrency
suspend fun fetchUserData(): User {
    // Suspends without blocking thread
    val user = apiService.getUser()
    return user
}

fun loadData() {
    viewModelScope.launch(Dispatchers.IO) {
        val user = fetchUserData() // Suspends, doesn't block
        withContext(Dispatchers.Main) {
            // Switch to main thread for UI update
            updateUI(user)
        }
    }
}

// Can create millions without overhead
repeat(100_000) {
    launch {
        delay(1000)
        println("Coroutine $it")
    }
}
```

### Threads:
```kotlin
// Heavy, preemptive concurrency
fun loadDataWithThread() {
    Thread {
        // Blocks the thread
        val user = apiService.getUser() // Blocking call
        
        runOnUiThread {
            updateUI(user)
        }
    }.start()
}

// Creating many threads is expensive
repeat(100_000) { // Would crash!
    Thread {
        Thread.sleep(1000)
        println("Thread $it")
    }.start()
}
```

**Key Differences:**
- **Coroutines**: Lightweight (thousands possible), suspendable, structured concurrency
- **Threads**: Heavy (limited number), blocking, manual lifecycle management

---

## 7. delay() vs Thread.sleep()

### delay() in Coroutines:
```kotlin
// Suspending function - doesn't block thread
suspend fun delayExample() {
    println("Before delay")
    delay(1000) // Suspends coroutine, thread can do other work
    println("After delay")
}

// Thread can handle other coroutines during delay
fun multipleCoroutines() {
    repeat(1000) {
        launch {
            delay(1000) // All can delay concurrently
            println("Coroutine $it finished")
        }
    }
}
```

### Thread.sleep() in Thread:
```kotlin
// Blocking function - blocks entire thread
fun sleepExample() {
    println("Before sleep")
    Thread.sleep(1000) // Blocks thread completely
    println("After sleep")
}

// Each thread blocks during sleep
fun multipleThreads() {
    repeat(10) { // Limited number due to thread overhead
        Thread {
            Thread.sleep(1000) // Thread is blocked, can't do other work
            println("Thread $it finished")
        }.start()
    }
}
```

**Key Differences:**
- **delay()**: Suspends coroutine, thread remains available for other work
- **Thread.sleep()**: Blocks entire thread, no other work can be done

---

## 8. SupervisorJob

### Regular Job:
```kotlin
fun regularJobExample() {
    val job = Job()
    
    launch(job) {
        launch {
            throw Exception("Child 1 failed")
        }
        launch {
            delay(2000)
            println("Child 2 completed") // Won't print - cancelled due to sibling failure
        }
    }
}
```

### SupervisorJob:
```kotlin
fun supervisorJobExample() {
    val supervisorJob = SupervisorJob()
    
    launch(supervisorJob) {
        launch {
            throw Exception("Child 1 failed")
        }
        launch {
            delay(2000) 
            println("Child 2 completed") // Will print - independent of sibling failure
        }
    }
}

// Practical example
class ViewModel : ViewModel() {
    private val supervisorJob = SupervisorJob()
    private val scope = CoroutineScope(Dispatchers.IO + supervisorJob)
    
    fun loadData() {
        // If one fails, others continue
        scope.launch {
            launch { loadUsers() }    // Independent
            launch { loadPosts() }    // Independent
            launch { loadComments() } // Independent
        }
    }
}
```

**Key Points:**
- **Regular Job**: Child failure cancels all siblings
- **SupervisorJob**: Children are independent, one failure doesn't affect others
- **Use Case**: When you want independent operations that shouldn't affect each other

---

## 9. Handler, Looper, and HandlerThread

### Handler:
```kotlin
// Sends messages/runnables to a thread's message queue
class MainActivity : AppCompatActivity() {
    private val handler = Handler(Looper.getMainLooper())
    
    fun updateUIAfterDelay() {
        handler.postDelayed({
            textView.text = "Updated after 2 seconds"
        }, 2000)
    }
    
    fun handleMessage() {
        val handler = object : Handler(Looper.getMainLooper()) {
            override fun handleMessage(msg: Message) {
                when (msg.what) {
                    1 -> textView.text = "Message 1"
                    2 -> textView.text = "Message 2"
                }
            }
        }
        
        // Send messages
        handler.sendEmptyMessage(1)
    }
}
```

### Looper:
```kotlin
// Message dispatcher for a thread
class BackgroundThread : Thread() {
    lateinit var handler: Handler
    
    override fun run() {
        Looper.prepare() // Prepare message queue for this thread
        
        handler = Handler(Looper.myLooper()!!) {
            // Handle messages on this background thread
            true
        }
        
        Looper.loop() // Start message processing
    }
}
```

### HandlerThread:
```kotlin
// Thread with built-in Looper
class NetworkManager {
    private val handlerThread = HandlerThread("NetworkThread")
    private lateinit var networkHandler: Handler
    
    init {
        handlerThread.start()
        networkHandler = Handler(handlerThread.looper)
    }
    
    fun performNetworkOperation() {
        networkHandler.post {
            // This runs on background thread
            val data = makeNetworkCall()
            
            // Switch back to main thread for UI update
            Handler(Looper.getMainLooper()).post {
                updateUI(data)
            }
        }
    }
    
    fun cleanup() {
        handlerThread.quitSafely()
    }
}
```

**Key Differences:**
- **Handler**: Sends/processes messages, associated with a specific thread's Looper
- **Looper**: Manages message queue for a thread, dispatches messages to Handler
- **HandlerThread**: Regular Thread + built-in Looper, convenient for background processing

**Relationship:**
```
Thread → has → Looper → dispatches to → Handler → processes → Messages/Runnables
```

**Modern Alternative:**
```kotlin
// Coroutines are preferred over Handler/Looper/HandlerThread
viewModelScope.launch(Dispatchers.IO) {
    val data = makeNetworkCall()
    withContext(Dispatchers.Main) {
        updateUI(data)
    }
}
```