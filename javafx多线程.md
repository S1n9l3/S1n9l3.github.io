---
article: false
title: FavaFX多线程
icon: any
order: 2
---



# FavaFX多线程

`javafx`运行时界面运行在主线程`javafx application`中，用于更新UI，但是如果执行了耗时的任务，比如网络请求，文件读写等操作，会导致线程阻塞，出现**界面无响应**的问题。

`javafx`提供了接口`Worker`，包括了三个类

* `ScheduledService`
* `Service`
* `Task`



## Task

假设需要一个应用：通过点击`开始`按钮执行任务，并有进度条来显示任务进度，并显示任务的相关信息，可以使用`取消`按钮来取消任务；

首先写一个简单的布局

```java
public class fxTask extends Application {
    public static void main(String[] args) {
        launch(args);
    }

    @Override
    public void start(Stage primaryStage) throws Exception {

        Button startBt = new Button("开始");
        Button cancelBt = new Button("取消");

        ProgressBar progressBar = new ProgressBar(0);
        progressBar.setPrefWidth(200);

        Label l1 = new Label("state");
        Label l2 = new Label("value");
        Label l3 = new Label("title");
        Label l4 = new Label("message");

        HBox hBox = new HBox();
        hBox.setAlignment(Pos.CENTER);
        hBox.setSpacing(10.0);

        hBox.getChildren().addAll(startBt, cancelBt, progressBar, l1, l2, l3, l4);

        AnchorPane anchorPane = new AnchorPane();
        anchorPane.setStyle("-fx-background-color: #7bb9b0;");
        anchorPane.getChildren().add(hBox);

        AnchorPane.setLeftAnchor(hBox, 100.0);
        AnchorPane.setTopAnchor(hBox, 100.0);

        Scene scene = new Scene(anchorPane);
        primaryStage.setScene(scene);
        primaryStage.setTitle("javaFX");
        primaryStage.setWidth(700);
        primaryStage.setHeight(500);
        primaryStage.show();
    }
}
```



![image-20220918113142047](assets/javafx%E5%A4%9A%E7%BA%BF%E7%A8%8B/image-20220918113142047.png)



定义一个任务类，继承`Task`，需要指定泛型来设置返回类型，

```java
class MyTask extends Task<Number> {

    @Override
    protected void updateProgress(long workDone, long max) {
        super.updateProgress(workDone, max);
        // 这里设置任务进度
    }

    @Override
    protected void updateProgress(double workDone, double max) {
        super.updateProgress(workDone, max);
        // 这里设置任务进度
    }

    @Override
    protected void updateMessage(String message) {
        super.updateMessage(message);
        // 这里设置任务中的信息
    }

    @Override
    protected void updateTitle(String title) {
        super.updateTitle(title);
        // 这里更新任务名
    }

    @Override
    protected void updateValue(Number value) {
        super.updateValue(value);
        // 在这里返回结果
        
    }

    @Override
    protected Number call() throws Exception {
        // 这里是任务主体
        
        return 0;
    }
}
```



在主程序中设置任务，绑定按钮点击事件，这样点击`开始`按钮后任务就开始了，

```java
MyTask myTask = new MyTask();

Thread thread = new Thread(myTask);

startBt.setOnAction(new EventHandler<ActionEvent>() {
    @Override
    public void handle(ActionEvent event) {
        thread.start();
    }
});
```



需要注意的是这里执行的时候是新开的一个线程，与UI相关的都是`FXApplication`线程完成的，即主线程；所以点击按钮后，一个新的线程执行任务，返回结果会到主线程中；在任务中和返回结果出打印线程，

```java
@Override
protected void updateValue(Number value) {
    super.updateValue(value);
    // 在这里返回结果
    // 这里执行是在fx线程，返回true
    System.out.println(Platform.isFxApplicationThread());
    System.out.println(value);
}

@Override
protected Number call() throws Exception {
    // 这里是任务主体
    // 判断是否是fx线程，这里返回的是false
    System.out.println(Platform.isFxApplicationThread());
}
```



设置这样的一个任务，模拟进度条，每次随机增长，直到达到设定值，

```java
@Override
protected Number call() throws Exception {
    // 这里是任务主体

    // 设置任务名称
    this.updateTitle("任务名称");

    double sum = 1000;
    Random rand = new Random();
    double cur = 0;
    double progress = 0;
    while (cur < sum){
        cur = cur + rand.nextInt(100);
        if (cur > sum){
            cur = sum;
        }
        progress = cur / sum;

        // 更新进度
        this.updateProgress(cur, sum);
		// 更新任务信息
        if (progress < 0.5){
            this.updateMessage("请耐性等待");
        } else if (progress < 0.8){
            this.updateMessage("马上就好");
        } else if (progress < 1){
            this.updateMessage("即将完成");
        } else if (progress >= 1){
            this.updateMessage("完成了!");
        }

        Thread.sleep(200);

    }
	// 这里返回了任务进度
    return progress;
}

```



对于任务设置监听事件，监听任务中的状态变化或者信息输出等；需要注意在任务类中重写方法与设置监听事件两者是相同的，但更推荐设置监听事件；

```java
// 任务进度监听，在任务中调用的 this.updateProgress(cur, sum) 会在这里处理
// this.updateProgress(cur, sum) 有两个参数：当前数和总数
// 而监听事件的值是两者的比值，表示进度
myTask.progressProperty().addListener(new ChangeListener<Number>() {
    @Override
    public void changed(ObservableValue<? extends Number> observable, Number oldValue, Number newValue) {
        progressBar.setProgress(newValue.doubleValue());

    }

});


// 当任务名称改变时会触发事件
// 在任务中调用了 this.updateTitle("任务名称") 会触发该事件
myTask.titleProperty().addListener(new ChangeListener<String>() {
    @Override
    public void changed(ObservableValue<? extends String> observable, String oldValue, String newValue) {
        l3.setText(newValue);
    }
});


// 任务结束后得到返回值
// 由于任务中返回的是进度，所以这里判断进度为1时输出"完成"
myTask.valueProperty().addListener(new ChangeListener<Number>() {
    @Override
    public void changed(ObservableValue<? extends Number> observable, Number oldValue, Number newValue) {
        if (newValue.doubleValue() == 1){
            l2.setText("完成");
        }
    }
});


// 设置任务执行中的信息会触发该事件
// 在任务中调用了 this.updateMessage("请耐性等待") 触发该事件
myTask.messageProperty().addListener(new ChangeListener<String>() {
    @Override
    public void changed(ObservableValue<? extends String> observable, String oldValue, String newValue) {
        l4.setText(newValue);
    }
});


// 任务状态监听事件
// 当状态改变时会触发该事件，任务状态包括
// * READY
// * SCHEDULED
// * RUNNING
// * SUCCEEDED
// * CANCELLED
// * FAILED
myTask.stateProperty().addListener(new ChangeListener<Worker.State>() {
    @Override
    public void changed(ObservableValue<? extends Worker.State> observable, Worker.State oldValue, Worker.State newValue) {
        System.out.println(newValue.toString());
        l1.setText(newValue.toString());
    }
});


// 任务异常监听事件
// 当任务未能正常完成时会触发该事件，取消事件也会调用该事件
myTask.exceptionProperty().addListener(new ChangeListener<Throwable>() {
    @Override
    public void changed(ObservableValue<? extends Throwable> observable, Throwable oldValue, Throwable newValue) {
        // 异常监听
        System.out.println("异常事件监听 " + newValue.getMessage());
    }
});
```



另外需要设置任务的取消，添加按钮绑定事件，

```java
cancelBt.setOnAction(new EventHandler<ActionEvent>() {
    @Override
    public void handle(ActionEvent event) {
        // 取消任务
        myTask.cancel();
    }
});
```



为了保证任务能够正常的取消，在任务中需要另外进行任务取消的判断，在任务循环中添加判断，

```java
if (isCancelled()){
    // 任务取消需要在这里另外判断一下
    break;
}
```



执行任务可以得到结果，

![image-20220918140836667](assets/javafx%E5%A4%9A%E7%BA%BF%E7%A8%8B/image-20220918140836667.png)

![image-20220918140843365](assets/javafx%E5%A4%9A%E7%BA%BF%E7%A8%8B/image-20220918140843365.png)

![image-20220918140859438](assets/javafx%E5%A4%9A%E7%BA%BF%E7%A8%8B/image-20220918140859438.png)



**特别注意**：`Task`设置的任务只能执行一次，如果再次点击`开始`会出错；如果执行任务时界面关闭了，那么任务仍然会执行；

全部代码，

```java
package javafxTask;

import javafx.application.Application;
import javafx.beans.value.ChangeListener;
import javafx.beans.value.ObservableValue;
import javafx.concurrent.Task;
import javafx.concurrent.Worker;
import javafx.event.ActionEvent;
import javafx.event.EventHandler;
import javafx.geometry.Pos;
import javafx.scene.Scene;
import javafx.scene.control.Button;
import javafx.scene.control.Label;
import javafx.scene.control.ProgressBar;
import javafx.scene.layout.AnchorPane;
import javafx.scene.layout.HBox;
import javafx.stage.Stage;

import java.util.Random;


public class fxTask extends Application {
    public static void main(String[] args) {
        launch(args);
    }

    @Override
    public void start(Stage primaryStage) throws Exception {

        Button startBt = new Button("开始");
        Button cancelBt = new Button("取消");

        ProgressBar progressBar = new ProgressBar(0);
        progressBar.setPrefWidth(200);

        Label l1 = new Label("state");
        Label l2 = new Label("value");
        Label l3 = new Label("title");
        Label l4 = new Label("message");

        HBox hBox = new HBox();
        hBox.setAlignment(Pos.CENTER);
        hBox.setSpacing(10.0);

        hBox.getChildren().addAll(startBt, cancelBt, progressBar, l1, l2, l3, l4);

        AnchorPane anchorPane = new AnchorPane();
        anchorPane.setStyle("-fx-background-color: #7bb9b0;");
        anchorPane.getChildren().add(hBox);

        AnchorPane.setLeftAnchor(hBox, 100.0);
        AnchorPane.setTopAnchor(hBox, 100.0);

        Scene scene = new Scene(anchorPane);
        primaryStage.setScene(scene);
        primaryStage.setTitle("javaFX");
        primaryStage.setWidth(700);
        primaryStage.setHeight(500);
        primaryStage.show();

        MyTask myTask = new MyTask();

        Thread thread = new Thread(myTask);

        startBt.setOnAction(new EventHandler<ActionEvent>() {
            @Override
            public void handle(ActionEvent event) {
                thread.start();
            }
        });


        cancelBt.setOnAction(new EventHandler<ActionEvent>() {
            @Override
            public void handle(ActionEvent event) {
                // 取消任务
                myTask.cancel();
            }
        });

        myTask.progressProperty().addListener(new ChangeListener<Number>() {
            @Override
            public void changed(ObservableValue<? extends Number> observable, Number oldValue, Number newValue) {
                // 进度
                System.out.println("任务进度监听 " + newValue.doubleValue());
                progressBar.setProgress(newValue.doubleValue());
            }
        });

        myTask.titleProperty().addListener(new ChangeListener<String>() {
            @Override
            public void changed(ObservableValue<? extends String> observable, String oldValue, String newValue) {
                // 标题
                l3.setText(newValue);
            }
        });

        myTask.valueProperty().addListener(new ChangeListener<Number>() {
            @Override
            public void changed(ObservableValue<? extends Number> observable, Number oldValue, Number newValue) {
                System.out.println("返回值监听 " + newValue.doubleValue());
                if (newValue.doubleValue() == 1){
                    l2.setText("完成");
                }
            }
        });

        myTask.messageProperty().addListener(new ChangeListener<String>() {
            @Override
            public void changed(ObservableValue<? extends String> observable, String oldValue, String newValue) {
                l4.setText(newValue);
            }
        });

        myTask.stateProperty().addListener(new ChangeListener<Worker.State>() {
            @Override
            public void changed(ObservableValue<? extends Worker.State> observable, Worker.State oldValue, Worker.State newValue) {
                System.out.println(newValue.toString());
                l1.setText(newValue.toString());
            }
        });

        myTask.exceptionProperty().addListener(new ChangeListener<Throwable>() {
            @Override
            public void changed(ObservableValue<? extends Throwable> observable, Throwable oldValue, Throwable newValue) {
                // 异常监听
                System.out.println("异常事件监听 " + newValue.getMessage());
            }
        });

    }

}


class MyTask extends Task<Number> {
    @Override
    protected Number call() throws Exception {
        // 这里是任务主体
        // 判断是否是fx线程，这里返回的是false
        // System.out.println(Platform.isFxApplicationThread());

        // 任务名称
        this.updateTitle("任务名称");

        double sum = 1000;
        Random rand = new Random();
        double cur = 0;
        double progress = 0;
        while (cur < sum){
            cur = cur + rand.nextInt(100);
            if (cur > sum){
                cur = sum;
            }
            progress = cur / sum;

            this.updateProgress(cur, sum);

            if (progress < 0.5){
                this.updateMessage("请耐性等待");
            } else if (progress < 0.8){
                this.updateMessage("马上就好");
            } else if (progress < 1){
                this.updateMessage("即将完成");
            } else if (progress >= 1){
                this.updateMessage("完成了!");
            }


            Thread.sleep(200);

            if (isCancelled()){
                // 任务取消需要在这里另外判断一下
                break;
            }

        }

        return progress;
    }
}
```



## Service

`Service`类比`Task`类会更加的灵活，可以对任务进行重置、重启等操作。

在`Task`中的布局总新增一些内容，

```java
public void start(Stage primaryStage) throws Exception {

    Button startBt = new Button("开始");
    Button cancelBt = new Button("取消");
    Button restartBt = new Button("重启");
    Button resetBt = new Button("重置");

    ProgressBar progressBar = new ProgressBar(0);
    progressBar.setPrefWidth(200);

    Label l1 = new Label("state");
    Label l2 = new Label("value");
    Label l3 = new Label("title");
    Label l4 = new Label("message");

    HBox hBox = new HBox();
    hBox.setAlignment(Pos.CENTER);
    hBox.setSpacing(10.0);

    hBox.getChildren().addAll(startBt, cancelBt, restartBt, resetBt, progressBar, l1, l2, l3, l4);

    AnchorPane anchorPane = new AnchorPane();
    anchorPane.setStyle("-fx-background-color: #7bb9b0;");
    anchorPane.getChildren().add(hBox);

    AnchorPane.setLeftAnchor(hBox, 100.0);
    AnchorPane.setTopAnchor(hBox, 100.0);

    Scene scene = new Scene(anchorPane);
    primaryStage.setScene(scene);
    primaryStage.setTitle("javaFX");
    primaryStage.setWidth(700);
    primaryStage.setHeight(500);
    primaryStage.show();
}
```

![image-20220918185050708](assets/javafx%E5%A4%9A%E7%BA%BF%E7%A8%8B/image-20220918185050708.png)



新建一个任务类，继承于`Service`，需要实现`createTask`方法，返回的是`Task`，所以需要实例化一个`Task`作为返回值，`Task`的内容与上一节相同。

```java
class MyService extends Service<Number>{
    @Override
    protected Task<Number> createTask() {
        Task<Number> task = new Task<Number>() {
            @Override
            protected Number call() throws Exception {

                // 任务名称
                this.updateTitle("任务名称");

                double sum = 1000;
                Random rand = new Random();
                double cur = 0;
                double progress = 0;
                while (cur < sum) {
                    cur = cur + rand.nextInt(100);
                    if (cur > sum) {
                        cur = sum;
                    }
                    progress = cur / sum;
                    // System.out.println(progress);

                    this.updateProgress(cur, sum);

                    if (progress < 0.5) {
                        this.updateMessage("请耐性等待");
                    } else if (progress < 0.8) {
                        this.updateMessage("马上就好");
                    } else if (progress < 1) {
                        this.updateMessage("即将完成");
                    } else if (progress >= 1) {
                        this.updateMessage("完成了!");
                    }


                    Thread.sleep(200);

                    if (isCancelled()){
                        // 任务取消需要在这里另外判断一下
                        break;
                    }
                }

                return progress;
            }
        };

        return task;
    }
}

```



然后就可以实例化任务类，并绑定按钮事件了，

```java
MyService myService = new MyService();

// 开始事件
startBt.setOnAction(new EventHandler<ActionEvent>() {
    @Override
    public void handle(ActionEvent event) {
        myService.start();
    }
});

// 取消事件
cancelBt.setOnAction(new EventHandler<ActionEvent>() {
    @Override
    public void handle(ActionEvent event) {
        myService.cancel();
    }
});

// 重启事件
restartBt.setOnAction(new EventHandler<ActionEvent>() {
    @Override
    public void handle(ActionEvent event) {
        myService.restart();
    }
});

// 重置事件
resetBt.setOnAction(new EventHandler<ActionEvent>() {
    @Override
    public void handle(ActionEvent event) {
        myService.reset();
        // 进度条归零
        progressBar.setProgress(0);
    }
});

```



对任务设置一些监听事件，

```java
// 任务进度监听
myService.progressProperty().addListener(new ChangeListener<Number>() {
    @Override
    public void changed(ObservableValue<? extends Number> observable, Number oldValue, Number newValue) {
        progressBar.setProgress(newValue.doubleValue());
    }
});

// 消息监听
myService.messageProperty().addListener(new ChangeListener<String>() {
    @Override
    public void changed(ObservableValue<? extends String> observable, String oldValue, String newValue) {
        if (newValue != null){
            l4.setText(newValue);
        }
    }
});

// 返回值监听
myService.valueProperty().addListener(new ChangeListener<Number>() {
    @Override
    public void changed(ObservableValue<? extends Number> observable, Number oldValue, Number newValue) {
        if (newValue != null && newValue.doubleValue() == 1){
            l2.setText("完成");
        }
    }
});
```



最终效果，可以通过`开始`按钮开始任务，`取消`按钮取消任务，`重启`按钮重启任务，`重置`按钮重置任务，取消的任务不能通过`开始`再次启动，需要通过`重启`，或者`重置`后`开始`。

![image-20220918190139631](assets/javafx%E5%A4%9A%E7%BA%BF%E7%A8%8B/image-20220918190139631.png)



`Service`还可以添加任务某个状态的监听事件（`Task`类也可以的），比如`Running`状态的监听事件，当任务进入`Running`状态或离开`Running`状态时会触发该事件，

```java
myService.runningProperty().addListener(new ChangeListener<Boolean>() {
    @Override
    public void changed(ObservableValue<? extends Boolean> observable, Boolean oldValue, Boolean newValue) {
        System.out.println("Running " + newValue);
    }
});
```

最后的代码，

```java
package javafxTask;

import javafx.application.Application;
import javafx.beans.value.ChangeListener;
import javafx.beans.value.ObservableValue;
import javafx.concurrent.Service;
import javafx.concurrent.Task;
import javafx.event.ActionEvent;
import javafx.event.EventHandler;
import javafx.geometry.Pos;
import javafx.scene.Scene;
import javafx.scene.control.Button;
import javafx.scene.control.Label;
import javafx.scene.control.ProgressBar;
import javafx.scene.layout.AnchorPane;
import javafx.scene.layout.HBox;
import javafx.stage.Stage;

import java.util.Random;

public class fxService extends Application {
    public static void main(String[] args) {
        launch(args);
    }

    @Override
    public void start(Stage primaryStage) throws Exception {

        Button startBt = new Button("开始");
        Button cancelBt = new Button("取消");
        Button restartBt = new Button("重启");
        Button resetBt = new Button("重置");

        ProgressBar progressBar = new ProgressBar(0);
        progressBar.setPrefWidth(200);

        Label l1 = new Label("state");
        Label l2 = new Label("value");
        Label l3 = new Label("title");
        Label l4 = new Label("message");

        HBox hBox = new HBox();
        hBox.setAlignment(Pos.CENTER);
        hBox.setSpacing(10.0);

        hBox.getChildren().addAll(startBt, cancelBt, restartBt, resetBt, progressBar, l1, l2, l3, l4);

        AnchorPane anchorPane = new AnchorPane();
        anchorPane.setStyle("-fx-background-color: #7bb9b0;");
        anchorPane.getChildren().add(hBox);

        AnchorPane.setLeftAnchor(hBox, 100.0);
        AnchorPane.setTopAnchor(hBox, 100.0);

        Scene scene = new Scene(anchorPane);
        primaryStage.setScene(scene);
        primaryStage.setTitle("javaFX");
        primaryStage.setWidth(700);
        primaryStage.setHeight(500);
        primaryStage.show();

        MyService myService = new MyService();

        // 开始事件
        startBt.setOnAction(new EventHandler<ActionEvent>() {
            @Override
            public void handle(ActionEvent event) {
                myService.start();
            }
        });

        // 取消事件
        cancelBt.setOnAction(new EventHandler<ActionEvent>() {
            @Override
            public void handle(ActionEvent event) {
                myService.cancel();
            }
        });

        // 重启事件
        restartBt.setOnAction(new EventHandler<ActionEvent>() {
            @Override
            public void handle(ActionEvent event) {
                myService.restart();
            }
        });

        // 重置事件
        resetBt.setOnAction(new EventHandler<ActionEvent>() {
            @Override
            public void handle(ActionEvent event) {
                myService.reset();
                progressBar.setProgress(0);
            }
        });

        // 任务进度监听
        myService.progressProperty().addListener(new ChangeListener<Number>() {
            @Override
            public void changed(ObservableValue<? extends Number> observable, Number oldValue, Number newValue) {
                progressBar.setProgress(newValue.doubleValue());
            }
        });
        // 消息监听
        myService.messageProperty().addListener(new ChangeListener<String>() {
            @Override
            public void changed(ObservableValue<? extends String> observable, String oldValue, String newValue) {
                if (newValue != null){
                    l4.setText(newValue);
                }
            }
        });
        // 返回值监听
        myService.valueProperty().addListener(new ChangeListener<Number>() {
            @Override
            public void changed(ObservableValue<? extends Number> observable, Number oldValue, Number newValue) {
                if (newValue != null && newValue.doubleValue() == 1){
                    l2.setText("完成");
                }
            }
        });

        myService.runningProperty().addListener(new ChangeListener<Boolean>() {
            @Override
            public void changed(ObservableValue<? extends Boolean> observable, Boolean oldValue, Boolean newValue) {
                System.out.println("Running " + newValue);
            }
        });

    }
}


class MyService extends Service<Number>{
    @Override
    protected Task<Number> createTask() {
        Task<Number> task = new Task<Number>() {
            @Override
            protected Number call() throws Exception {

                // 任务名称
                this.updateTitle("任务名称");

                double sum = 1000;
                Random rand = new Random();
                double cur = 0;
                double progress = 0;
                while (cur < sum) {
                    cur = cur + rand.nextInt(100);
                    if (cur > sum) {
                        cur = sum;
                    }
                    progress = cur / sum;
                    // System.out.println(progress);

                    this.updateProgress(cur, sum);

                    if (progress < 0.5) {
                        this.updateMessage("请耐性等待");
                    } else if (progress < 0.8) {
                        this.updateMessage("马上就好");
                    } else if (progress < 1) {
                        this.updateMessage("即将完成");
                    } else if (progress >= 1) {
                        this.updateMessage("完成了!");
                    }


                    Thread.sleep(200);

                    if (isCancelled()){
                        // 任务取消需要在这里另外判断一下
                        break;
                    }
                }

                return progress;
            }
        };

        return task;
    }
}

```





## ScheduledService

`ScheduledService`是继承于`Service`的类，`Service`和`Task`都是实现`Worker`接口的类，`Task`继承了`FutureTask`。`ScheduledService`在`Service`的基础上，增加了计划任务，可以管理任务延时执行、周期执行等操作。

界面仍然使用`Service`的，这里同样需要重写`createTask`方法，，设置任务当点击`开始`后，等`3秒`后计数加`1`，每隔`1秒`执行一次任务，当计数为`5`的倍数时，停止任务，点击`取消`暂停计数，点击`重启`重新开始计数；

```java
class MyScheduledService extends ScheduledService<Number>{

    int sum = 0;

    @Override
    protected Task<Number> createTask() {
        Task<Number> task = new Task<Number>() {
            @Override
            protected Number call() throws Exception {

                sum = sum + 1;
                System.out.println(sum);

                return sum;
            }
        };

        return task;
    }
}

```



然后实例化任务，并进行一些设置，可以查看手册有更多的设置，

```java
MyScheduledService myScheduledService = new MyScheduledService();
// 任务开始后延迟执行
myScheduledService.setDelay(Duration.seconds(3));
// 周期执行
myScheduledService.setPeriod(Duration.seconds(1));
// 任务失败是否重试
myScheduledService.setRestartOnFailure(true);
// 最大失败重试次数
myScheduledService.setMaximumFailureCount(5);
```

其他按钮绑定事件和监听事件类似，

```java
startBt.setOnAction(new EventHandler<ActionEvent>() {
    @Override
    public void handle(ActionEvent event) {
        myScheduledService.start();
    }
});

cancelBt.setOnAction(new EventHandler<ActionEvent>() {
    @Override
    public void handle(ActionEvent event) {
        myScheduledService.cancel();
    }
});

restartBt.setOnAction(new EventHandler<ActionEvent>() {
    @Override
    public void handle(ActionEvent event) {
        myScheduledService.restart();
    }
});

resetBt.setOnAction(new EventHandler<ActionEvent>() {
    @Override
    public void handle(ActionEvent event) {
        myScheduledService.reset();
    }
});
```



```java
myScheduledService.valueProperty().addListener(new ChangeListener<Number>() {
    @Override
    public void changed(ObservableValue<? extends Number> observable, Number oldValue, Number newValue) {
        if (newValue != null){
            l2.setText(""+newValue);
        }
    }
});

myScheduledService.lastValueProperty().addListener(new ChangeListener<Number>() {
    @Override
    public void changed(ObservableValue<? extends Number> observable, Number oldValue, Number newValue) {
        System.out.println("lastValue " + newValue);
    }
});

myScheduledService.stateProperty().addListener(new ChangeListener<Worker.State>() {
    @Override
    public void changed(ObservableValue<? extends Worker.State> observable, Worker.State oldValue, Worker.State newValue) {
        System.out.println("当前状态 " + newValue);

    }
});

myScheduledService.valueProperty().addListener(new ChangeListener<Number>() {
    @Override
    public void changed(ObservableValue<? extends Number> observable, Number oldValue, Number newValue) {
        System.out.println(newValue);
        if (newValue != null && newValue.intValue()%5 == 0){
            myScheduledService.cancel();
            System.out.println("任务取消");
        }
    }
});
```



最后代码，这里没写`重置`，可以设置任务类中通过`set`和`get`方法设置进度归零，

```java
package javafxTask;

import javafx.application.Application;
import javafx.beans.value.ChangeListener;
import javafx.beans.value.ObservableValue;
import javafx.concurrent.ScheduledService;
import javafx.concurrent.Task;
import javafx.concurrent.Worker;
import javafx.event.ActionEvent;
import javafx.event.EventHandler;
import javafx.geometry.Pos;
import javafx.scene.Scene;
import javafx.scene.control.Button;
import javafx.scene.control.Label;
import javafx.scene.control.ProgressBar;
import javafx.scene.layout.AnchorPane;
import javafx.scene.layout.HBox;
import javafx.stage.Stage;
import javafx.util.Duration;

public class fxScheduledService extends Application {
    public static void main(String[] args) {
        launch(args);
    }

    @Override
    public void start(Stage primaryStage) throws Exception {
        Button startBt = new Button("开始");
        Button cancelBt = new Button("取消");
        Button restartBt = new Button("重启");
        Button resetBt = new Button("重置");

        ProgressBar progressBar = new ProgressBar(0);
        progressBar.setPrefWidth(200);

        Label l1 = new Label("state");
        Label l2 = new Label("value");
        Label l3 = new Label("title");
        Label l4 = new Label("message");

        HBox hBox = new HBox();
        hBox.setAlignment(Pos.CENTER);
        hBox.setSpacing(10.0);

        hBox.getChildren().addAll(startBt, cancelBt, restartBt, resetBt, progressBar, l1, l2, l3, l4);

        AnchorPane anchorPane = new AnchorPane();
        anchorPane.setStyle("-fx-background-color: #7bb9b0;");
        anchorPane.getChildren().add(hBox);

        AnchorPane.setLeftAnchor(hBox, 100.0);
        AnchorPane.setTopAnchor(hBox, 100.0);

        Scene scene = new Scene(anchorPane);
        primaryStage.setScene(scene);
        primaryStage.setTitle("javaFX");
        primaryStage.setWidth(700);
        primaryStage.setHeight(500);
        primaryStage.show();

        MyScheduledService myScheduledService = new MyScheduledService();
        // 任务开始后延迟执行
        myScheduledService.setDelay(Duration.seconds(3));
        // 周期执行
        myScheduledService.setPeriod(Duration.seconds(1));
        // 任务失败是否重试
        myScheduledService.setRestartOnFailure(true);
        // 最大失败次数
        myScheduledService.setMaximumFailureCount(5);

        startBt.setOnAction(new EventHandler<ActionEvent>() {
            @Override
            public void handle(ActionEvent event) {
                myScheduledService.start();
            }
        });

        cancelBt.setOnAction(new EventHandler<ActionEvent>() {
            @Override
            public void handle(ActionEvent event) {
                myScheduledService.cancel();
            }
        });

        restartBt.setOnAction(new EventHandler<ActionEvent>() {
            @Override
            public void handle(ActionEvent event) {
                myScheduledService.restart();
            }
        });

        resetBt.setOnAction(new EventHandler<ActionEvent>() {
            @Override
            public void handle(ActionEvent event) {
                myScheduledService.reset();
            }
        });

        ///////////
        myScheduledService.valueProperty().addListener(new ChangeListener<Number>() {
            @Override
            public void changed(ObservableValue<? extends Number> observable, Number oldValue, Number newValue) {
                if (newValue != null){
                    l2.setText(""+newValue);
                }
            }
        });

        myScheduledService.lastValueProperty().addListener(new ChangeListener<Number>() {
            @Override
            public void changed(ObservableValue<? extends Number> observable, Number oldValue, Number newValue) {
                System.out.println("lastValue " + newValue);
            }
        });

        myScheduledService.stateProperty().addListener(new ChangeListener<Worker.State>() {
            @Override
            public void changed(ObservableValue<? extends Worker.State> observable, Worker.State oldValue, Worker.State newValue) {
                System.out.println("当前状态 " + newValue);

            }
        });

        myScheduledService.valueProperty().addListener(new ChangeListener<Number>() {
            @Override
            public void changed(ObservableValue<? extends Number> observable, Number oldValue, Number newValue) {
                System.out.println(newValue);
                if (newValue != null && newValue.intValue()%5 == 0){
                    myScheduledService.cancel();
                    System.out.println("任务取消");
                }
            }
        });

    }
}

class MyScheduledService extends ScheduledService<Number>{

    int sum = 0;

    @Override
    protected Task<Number> createTask() {
        Task<Number> task = new Task<Number>() {
            @Override
            protected Number call() throws Exception {

                sum = sum + 1;
                System.out.println(sum);

                return sum;
            }
        };

        return task;
    }
}

```



## 创建多任务同时开始并返回值

有这样的任务：点击按钮，同时开始多个任务，并在任务结束后得到返回值。

```java
package javafxTask;

import javafx.application.Application;
import javafx.beans.value.ChangeListener;
import javafx.beans.value.ObservableValue;
import javafx.concurrent.Service;
import javafx.concurrent.Task;
import javafx.event.ActionEvent;
import javafx.event.EventHandler;
import javafx.geometry.Pos;
import javafx.scene.Scene;
import javafx.scene.control.Button;
import javafx.scene.control.TextArea;
import javafx.scene.layout.AnchorPane;
import javafx.scene.layout.VBox;
import javafx.stage.Stage;

public class fxService_1 extends Application {
    public static void main(String[] args) {
        launch(args);
    }

    @Override
    public void start(Stage primaryStage) throws Exception {

        Button button = new Button("按钮");
        TextArea textArea = new TextArea();
        textArea.setPrefWidth(200);
        textArea.setPrefHeight(200);

        VBox vBox = new VBox();
        vBox.getChildren().addAll(button, textArea);
        vBox.setSpacing(10.0);
        vBox.setAlignment(Pos.CENTER);

        AnchorPane anchorPane = new AnchorPane();
        anchorPane.getChildren().add(vBox);
        anchorPane.setStyle("-fx-background-color: #c1da87;");

        Scene scene = new Scene(anchorPane);
        primaryStage.setScene(scene);
        primaryStage.setWidth(300);
        primaryStage.setHeight(300);
        primaryStage.show();

        button.setOnAction(new EventHandler<ActionEvent>() {
            @Override
            public void handle(ActionEvent event) {
                for (int i = 0; i < 10; i++) {
                    MultiTask1 multiTask1 = new MultiTask1("任务"+i);
                    multiTask1.start();
                    multiTask1.valueProperty().addListener(new ChangeListener<String[]>() {
                        @Override
                        public void changed(ObservableValue<? extends String[]> observable, String[] oldValue, String[] newValue) {
                            for (String s : newValue) {
                                textArea.appendText(s + "\t");
                            }
                            textArea.appendText("\n");
                        }
                    });
                }

            }
        });

    }

}


class MultiTask1 extends Service<String[]>{

    private final String name;

    public MultiTask1(String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }

    @Override
    protected Task<String[]> createTask() {

        Task<String[]> task = new Task<String[]>() {
            @Override
            protected String[] call() throws Exception {

                System.out.println(name + " 开始 " + Thread.currentThread().getName());

                Thread.sleep(5000);

                System.out.println(name + " 结束 " + Thread.currentThread().getName());

                return new String[]{name, "true"};
            }
        };

        return task;
    }
}

```



![image-20220918215324960](assets/javafx%E5%A4%9A%E7%BA%BF%E7%A8%8B/image-20220918215324960.png)