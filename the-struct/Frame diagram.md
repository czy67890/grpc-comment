# Server的工作方式
在grpc中存在三种Server，一种是用C语言实现的Server(仅仅是一个typedef后文会讲到这种设计),第二种是grpc core中的server，第三种是grpc提供给外部使用的Server，第三种Server中包含着第一种(同时也是第二种)Server的指针.并且基于该指针完成功能。
 ```cpp
typedef struct grpc_server grpc_server;
```




这样的typedef在GRPC中使用非常广泛，GRPC中实现了一个CPPIMPL的模板类，该类提供了一个FromC的功能，能够将GRPC中的C声明的语法与CPP语言的强大的面向对象功能一起使用
```cpp
template <typename CppType, typename CType>
class CppImplOf {
 public:
  // Convert the C struct to C++
  static CppType* FromC(CType* c_type) {
    return reinterpret_cast<CppType*>(c_type);
  }

  static const CppType* FromC(const CType* c_type) {
    return reinterpret_cast<const CppType*>(c_type);
  }

  // Retrieve a c pointer (of the same ownership as this)
  CType* c_ptr() {
    return reinterpret_cast<CType*>(static_cast<CppType*>(this));
  }

 protected:
  ~CppImplOf() = default;
};

```
而这样只需要使用到类似于Server这样的声明，而不需要给出定义，实际上给出struct server的定义在这样的模板技巧下造成的后果是ub的，grpc中大多数的类都给出了c语言的api接口，而核心的实现则是通过CPP来封装与实现，即给出了两套不同的调用手段。

回到一开始的Server的架构设计，Server在grpc和grpc core中各给出了一套定义，grpc core中的Server是内部使用的Server定义，而在grpcpp中定义定义的Server则是作为类库的实现暴露给开发者。

定义在core中的Server继承了两个类，继承代码给出如下
```cpp
class Server : public ServerInterface,
               public InternallyRefCounted<Server>,
               public CppImplOf<Server, grpc_server> 
```
其中的InternallyRefCounted的定义说明如下,内部有一个RefCount类的成员，Orphanable实际上是一个接口，暴露一个Orphan接口供子类复写，而InternallyRefCounted则是一个类似智能指针的类，不过是通过继承来实现功能，并且通过Ref与Unref操作来手动的维护计数。 
```cpp
template <typename Child, typename UnrefBehavior = UnrefDelete>
class InternallyRefCounted : public Orphanable
```
暴露给库的使用者的Server有如下的定义
```cpp
class Server : public ServerInterface, private internal::GrpcLibrary
```
这样的设计使得接口与实现实际上是分离的，而groc core中的ServerInterface与grpc的ServerInterface并不是一个类
grpc中定义的ServerInterface继承关系如下：
```cpp
class ServerInterface : public internal::CallHook
```
grpc core中的Server Interface









