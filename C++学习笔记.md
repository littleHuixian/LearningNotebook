C++学习笔记

# 栈模板

```c++
#include<iostream>
//01-模版类-栈
class Stack{
private:
    int* items;      //栈数组
    int stacksize;   //栈实际大小
    int top;         //栈顶指针
public:
    //构造函数 1.分配栈数组内存； 2.把栈顶指针初始化为0  
    Stack(int size):stacksize(size),top(0){
        items = new int[stacksize];
    }
    //析构函数：释放内存
    ~Stack(){
        delete []items;
        items = nullptr;
    }
    //判断栈是否为空
    bool isempty() const {
        if (top == 0) return true;
        return false;
        //return top==0;
    }
    //判断栈是否已满
    bool isfull() const {
        return top == stacksize;
    }
    //元素入栈
    bool push(const int& item) {
        if(top < stacksize){
            items[top++]=item;
            return true;
        }
        return false;
    }
    //元素出栈
    bool pop(int& item) {
        if(top>0){
            item=items[--top];
            return true;
        }
        return false;
    }
};
int main(){
    Stack ss(5);
    //存入int型数据
    ss.push(1);
    ss.push(2);
    ss.push(4);
    ss.push(6);
    //元素出栈
    int item;
    while(ss.isempty()==false){
        ss.pop(item);
        std::cout<<"item=="<<item<<std::endl;
    }
    return 0;
}
```

在实际应用场景中，数据的类型有多种，则需要使用模板

```c++
//typedef std::string DataType;    //根据不同的数据类型进行改动此处
template<class DataType>
class Stack{
private:
    DataType* items;      //栈数组
    int stacksize;   //栈实际大小
    int top;         //栈顶指针
public:
    //构造函数 1.分配栈数组内存； 2.把栈顶指针初始化为0  
    Stack(int size):stacksize(size),top(0){
        items = new DataType[stacksize];
    }
    //析构函数：释放内存
    ~Stack(){
        delete []items;
        items = nullptr;
    }
    //判断栈是否为空
    bool isempty() const {
        if (top == 0) return true;
        return false;
        //return top==0;
    }
    //判断栈是否已满
    bool isfull() const {
        return top == stacksize;
    }
    //元素入栈
    bool push(const DataType& item) {
        if(top < stacksize){
            items[top++]=item;
            return true;
        }
        return false;
    }
    //元素出栈
    bool pop(DataType& item) {
        if(top>0){
            item=items[--top];
            return true;
        }
        return false;
    }
};
```

在调用模板类的使用时，需要指定数据类型

```c++
int main(){
    Stack<string> ss(5);
    //存入int型数据
    ss.push('eeee');
    ss.push('bbbb');
    ss.push('ffff');
    ss.push('yyyy');
    //元素出栈
    DataType item;
    while(ss.isempty()==false){
        ss.pop(item);
        std::cout<<"item=="<<item<<std::endl;
    }
    return 0;
}
```

