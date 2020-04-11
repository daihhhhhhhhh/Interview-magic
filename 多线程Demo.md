# 1、三个售票窗口同时出售20张票

程序分析：

（1）票数要使用同一个静态值

（2）为保证不会出现卖出同一个票数，要java多线程同步锁

设计思路：

（1）创建一个站台类Station，继承Thread，重写run方法，在run方法里面执行售票操作！售票员要使用同步锁：即有一个站台卖出这张票时，其他站台要等这张票买完！

```java
public class Station extends Thread{
    //通过构造方法给线程名字赋值
    public Station(String name){
        //给线程名字赋值
        super(name);
    }
    //为了保证票数一直，票数要静态
    static int tick = 20;
    //创建一个静态钥匙
    static Object ob = "aa";
    //重写run方法
    @Override
    public void run(){
        while(tick > 0){
            //这个很重要，必须要使用一个锁
            synchronized(ob){
                if(tick > 0){
                    System.out.println(getName() + "卖出了第" + tick + "张票");
                    tick--;                        
                }else{
                    System.out.println("票卖完了");
                }
            }
            try{
                //休息1s
                sleep(1000);
            }catch(InterrupterException){
                e.printStackTrace();
            }
        }
    }
}
```

(2)创建主方法调用类

```java
public class MainClass{
    public static void main(String[] args){
        //实例化站台对象，并为每一个站台取名字
        Station station1 = new Station("窗口1");
        Station station2 = new Station("窗口2");
        Station station3 = new Station("窗口3");
        //让每一个站台对象各自开始工作
        station1.start();
        station2.start();
        station3.start();
    }
}
```

```
运行结果

窗口1卖出了第20张票
窗口2卖出了第19张票
窗口3卖出了第18张票
窗口3卖出了第17张票
窗口1卖出了第16张票
窗口2卖出了第15张票
窗口3卖出了第14张票
窗口1卖出了第13张票
窗口2卖出了第12张票
窗口2卖出了第11张票
窗口1卖出了第10张票
窗口3卖出了第9张票
窗口3卖出了第8张票
窗口1卖出了第7张票
窗口2卖出了第6张票
窗口3卖出了第5张票
窗口1卖出了第4张票
窗口2卖出了第3张票
窗口3卖出了第2张票
窗口1卖出了第1张票
票卖完了

```

# 2、两个人AB通过一个账户，A在柜台取钱，B在ATM取钱

程序分析：

钱的数量要设置成一个静态的变量，两个人要取得同一个对象值。

（1）创建一个Bank()类

```java
public class Bank{
	///假设一个账户有1000块钱
    static double money = 1000;
    //柜台Counter的取钱方法
    private void Counter(double money){
        Bank.money -= money;
        System.out.println("柜台取钱" + money + "元，还剩" + Bank.money + "元！");
	}
	//ATM的取钱方法
    private void ATM(double money){
        Bank.money -= money;
        System.out.println("ATM取钱" + money + "元，还剩" + Bank.money + "元！");
	}
    public synchronized void out Money(double money,String mode) throw Exception{
        if(money > Bank.money){
            throw new Exception("取款金额" + money + ",余额只剩" + Bank.money + "，取款失败！");
        }
        if(Object.equals(mode,"ATM")){
            ATM(money);
        }else{
            Counter(money);
        }
    }
}
```

（2）创建一个PersonA类

```java
public class PersonA extends Thread{
    Bank bank;
    String mode;
    public PersonA(Bank bank,String mode){
        this.bank = bank;
        this.mode = mode;
    }
    @Override
    public void run(){
        while(bank.money >= 100){
            try{
                bank.outMoney(100,mode);
            }catch (Exception e){
                e.printStackTrace();
            }
            try{
                sleep(1000);
            }catch(InterruptedException){
                e.printStackTrace();
            }
        }
    }
}
```

（3）创建一个PersonB类

```java
public class PersonB extends Thread{
    Bank bank;
    String mode;
    public PersonB(Bank bank,String mode){
        this.bank = bank;
        this.mode = mode;
    }
    @Override
    public void run(){
        while(bank.money >= 200){
            try{
                bank.outMoney(200,mode);
            }catch (Exception e){
                e.printStackTrace();
            }
            try{
                sleep(1000);
            }catch(InterruptedException){
                e.printStackTrace();
            }
        }
    }
}
```

（4）创建主方法的调用类

```java
public class MainClass{
    public static void main(String[] args){
        Bank bank = new Bank();
        PersonA personA = new PersonA(bank,"ATM");
        PersonB personB = new PersonB(bank,"Counter");
        personA.start();
        personB.start();
    }
}
```

