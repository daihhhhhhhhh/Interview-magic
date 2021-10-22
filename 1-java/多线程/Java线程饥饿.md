# Java线程饥饿

当连续不断拒绝线程访问资源并因此无法取得进展时，就会发生饥饿。当贪婪的线程长时间使用共享资源时，通常会发生这种情况。如果这种情况持续了很长时间，则线程没有获得足够的CPU时间或对资源的访问将无法取得足够的进展，从而导致线程不足。线程饥饿的可能原因之一是不同线程或线程组之间的线程优先级不正确。

另一个可能的原因可能是使用非终止循环（无限循环）或在特定资源上等待过多时间，同时保留了其他线程所需的关键锁。

通常建议尽量避免修改线程优先级，因为这是导致线程饥饿的主要原因。一旦开始使用线程优先级调整应用程序，它就会与特定平台紧密耦合，并且还会带来线程匮乏的风险。


Java线程饥饿示例
在我的示例中，我将总共创建五个线程。每个线程将被分配一个不同的线程优先级。创建线程并分配优先级后，我们将继续并启动所有五个线程。在主线程中，我们将等待5000ms或5秒，并将isActive标志更改为false，以便所有线程退出while循环并进入死线程状态。

实现Runnable接口的Worker类正在互斥对象（对象）上进行同步，以模拟线程锁定代码的关键部分，即使我确实对AtomicInteger使用并发类也可以执行getAndIncrement操作并且不需要锁定。我正在使用一个计数器，以便我们可以计数并查看每个工作线程执行工作的频率。作为一般准则，优先级较高的线程应获得更多的CPU周期，因此，优先级较高的线程的值应更大。

注意
Windows实现了一种线程回退机制，通过该机制，可以使长时间没有机会运行的线程获得临时的优先级提升，因此几乎无法实现完全的饥饿。但是，从我生成的数字中，您可以看到线程优先级如何对分配给线程5的CPU时间量有相当大的影响。

线程饥饿示例

```java
package com.example.thread.lockstarvation;

public class BankAccount {
	private double balance;
	int id;

	BankAccount(int id, double balance) {
		this.id = id;
		this.balance = balance;
	}
	 
	synchronized double getBalance() {
		// Wait to simulate io like database access ...
		try {
			Thread.sleep(100l);
		} catch (InterruptedException e) {
		}
		return balance;
	}
	 
	synchronized void withdraw(double amount) {
		balance -= amount;
	}
	 
	synchronized void deposit(double amount) {
		balance += amount;
	}
	 
	synchronized void transfer(BankAccount to, double amount) {
		withdraw(amount);
		to.deposit(amount);
	}
	 
	public static void main(String[] args) {
		final BankAccount fooAccount = new BankAccount(1, 500d);
		final BankAccount barAccount = new BankAccount(2, 500d);
	 
		Thread balanceMonitorThread1 = new Thread(new BalanceMonitor(fooAccount), "BalanceMonitor");
		Thread transactionThread1 = new Thread(new Transaction(fooAccount, barAccount, 250d), "Transaction-1");
		Thread transactionThread2 = new Thread(new Transaction(fooAccount, barAccount, 250d), "Transaction-2");
	 
		balanceMonitorThread1.setPriority(Thread.MAX_PRIORITY);
		transactionThread1.setPriority(Thread.MIN_PRIORITY);
		transactionThread2.setPriority(Thread.MIN_PRIORITY);
	 
		// Start the monitor
		balanceMonitorThread1.start();
	 
		// And later, transaction threads tries to execute.
		try {
			Thread.sleep(100l);
		} catch (InterruptedException e) {
		}
		transactionThread1.start();
		transactionThread2.start();
	 
	}

}

class BalanceMonitor implements Runnable {
	private BankAccount account;

	BalanceMonitor(BankAccount account) {
		this.account = account;
	}
	 
	boolean alreadyNotified = false;
	 
	@Override
	public void run() {
		System.out.format("%s started execution%n", Thread.currentThread().getName());
		while (true) {
			if (account.getBalance() <= 0) {
				// send email, or sms, clouds of smoke ...
				break;
			}
		}
		System.out.format("%s : account has gone too low, email sent, exiting.", Thread.currentThread().getName());
	}

}

class Transaction implements Runnable {
	private BankAccount sourceAccount, destinationAccount;
	private double amount;

	Transaction(BankAccount sourceAccount, BankAccount destinationAccount, double amount) {
		this.sourceAccount = sourceAccount;
		this.destinationAccount = destinationAccount;
		this.amount = amount;
	}
	 
	public void run() {
		System.out.format("%s started execution%n", Thread.currentThread().getName());
		sourceAccount.transfer(destinationAccount, amount);
		System.out.printf("%s completed execution%n", Thread.currentThread().getName());
	}

}

```

