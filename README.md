[合集 \- C\+\+(1\)](https://github.com)1\.C\+\+11 线程同步接口std::condition\_variable和std::future的简单使用09\-17收起
## std::condition\_variable



条件变量std::condition\_variable有wait和notify接口用于线程间的同步。如下图所示，Thread 2阻塞在wait接口，Thread 1通过notify接口通知Thread 2继续执行。
![con_variable_result](https://img2024.cnblogs.com/blog/1306820/202409/1306820-20240917004424233-2001547174.png)


具体参见示例代码：




```
#include
#include
#include
#include
std::mutex mt;
std::queue<int> data;
std::condition_variable cv;
auto start=std::chrono::high_resolution_clock::now();

void logCurrentTime()
{
	auto end = std::chrono::high_resolution_clock::now();
	auto elapsed = std::chrono::duration_cast<std::chrono::milliseconds>(end - start).count();
	std::cout << elapsed << ":";
}
void prepare_data()
{	
	logCurrentTime();
	std::cout << "this is " << __FUNCTION__ << " thread:" << std::this_thread::get_id() << std::endl;
	for (int i = 0; i < 10; i++)
	{
		data.push(i);
		logCurrentTime();
		std::cout << "data OK:" << i << std::endl;
	}
	//start to notify consume_data thread data is OK!
	cv.notify_one();
}


void consume_data()
{
	logCurrentTime();
	std::cout << "this is: " << __FUNCTION__ << " thread:" << std::this_thread::get_id() << std::endl;
	std::unique_lock<std::mutex> lk(mt);
	//wait first for notification
	cv.wait(lk);  //it must accept a unique_lock parameter to wait

	while (!data.empty())
	{
		logCurrentTime();
		std::cout << "data consumed: " << data.front() << std::endl;
		data.pop();
	}
}


int main()
{
	std::thread t2(consume_data);
	//wait for a while to wait first then prepare data,otherwise stuck on wait
	std::this_thread::sleep_for(std::chrono::milliseconds(10));
	std::thread t1(prepare_data);
	t1.join();
	t2.join();
	return 0;
}

```

### 输出结果


![con_variable_result](https://img2024.cnblogs.com/blog/1306820/202409/1306820-20240917004624021-978839802.png)


### 分析



主线程中另启两个线程，分别执行consume\_data和prepare\_data，其中consume\_data要先执行，以保证先等待再通知，否则若先通知再等待就死锁了。首先consume\_data线程在从**wait** 处阻塞等待。后prepare\_data线程中依次向队列写入0\-10，写完之后通过**notify\_one** 通知consume\_data线程解除阻塞，依次读取0\-10。



## std::future



std::future与std::async配合异步执行代码，再通过wait或get接口阻塞当前线程等待结果。如下图所示，Thread 2中future接口的get或wait接口会阻塞当前线程，std::async异步开启的新线程Thread1执行结束后，将结果存于std::future后通知Thread 1获取结果后继续执行.
![con_variable_result](https://img2024.cnblogs.com/blog/1306820/202409/1306820-20240917004637698-1845654580.png)


具体参见如下代码：




```
#include 
#include 
#include

int test()
{
	std::cout << "this is " << __FUNCTION__ << " thread:" << std::this_thread::get_id() << std::endl;;
	std::this_thread::sleep_for(std::chrono::microseconds(1000));
	return 10;
}
int main()
{
	std::cout << "this is " <<__FUNCTION__<<" thread:" << std::this_thread::get_id() << std::endl;;
	//this will lanuch on another thread
	std::future<int> result = std::async(test);

	std::cout << "After lanuch a thread: "<< std::this_thread::get_id() << std::endl;

	//block the thread and wait for the result
	std::cout << "result is: " <std::endl;

	std::cout << "After get result "<< std::endl;

	return 0;
}

```

### 输出结果


![运行结果](https://img2024.cnblogs.com/blog/1306820/202409/1306820-20240917004655243-267102588.png)


### 分析



主程序中调用std::async异步调用test函数，可以看到main函数的线程ID 27428与test函数执行的线程ID 9704并不一样，说明std::async另起了一个新的线程。在test线程中，先sleep 1000ms，所以可以看到"After lanuch a thread:"先输出，说明主线程异步执行，不受子线程影响。而"After get result "最后输出，说明get()方法会阻塞主线程，直到获取结果。

 本博客参考[楚门加速器p](https://tianchuang88.com)。转载请注明出处！
