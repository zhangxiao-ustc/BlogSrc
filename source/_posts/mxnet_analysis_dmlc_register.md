----
title: MXNet 源码分析(dmlc_core/include/dmlc/register.h)
----

register.h主要提供对单例注册的支持

## class Registry
class Registry是一个singleton class
```c++
template<typename EntryType>
class Registry {
 public:
  ...
  inline EntryType &__REGISTER__(cosnt std::string& name) {
    CHECK_EQ(fmap_.const(name), 0U)
        << name << "already registered";
    EntryType *e = new EntryType();
    e->name = name;
    fmap_[name] = e;
    ...
    entry_list_.push_back(e);
    return *e;
  }
  static Registry *Get();
 private:
  std::vector<EntryType*> entry_list_;
  ...
  std::map<std::string, EntryType*> fmap_;
  Registry() {}

  ~Registry() {
    for (size_t i = 0; i < entry_list_.size(); ++i) {
      delete entry_list_[i];
    }
  }
};
...
#define DMLC_REGISTRY_ENABLE(EntryType)                   \
  template<>                                              \
  Registry<EntryType > * Registry<EntryType >::Get() {    \
    static Registry<EntryType > inst;                     \
    return &inst;                                         \
  }
```
将ctor置于private中，禁止外部调用，singleton Registry只能通过Get()获得,有趣的是dtor也被声明为private，那么Registry的资源如何收回？又在何时收回？

另一点是Get()的实现实际上并不在register.h中，而是出现在包含宏DMLC_REGISTRY_ENABLE的文件中，对于那些不含有该宏的方法，调用Get()会导致编译期报错，从而达到ENABLE/DISABLE的效果。

由于Registry是一个template class，所以EntryType不同，对应的singleton也不是同一个。

## class FunctionRegEntryBase
```c++
template<typename EntryType, typename FunctionType>
class FunctionRegEntryBase {
 public:
  std::string name;
  ...
  FunctionType body;
  std::string return_type;

  inline EntryType &set_body(FunctionType body) {
    this->body = body;
    return this->self();
  }
  ...
  inline EntryType &set_return_type(const std::string &type) {
    return_type = type;
    return this->self();
  }

 protected:
  inline EntryType &self() {
    return *(static_cast<EntryType*>(this));
  }
};
```
新奇的写法，FunctionType还可以通过template deduction得出，EntryType只出现在返回值上，而相应的return位置上也没有明确的类型，似乎只能通过显式实例化来确定。
再有，单就该函数看来，FunctionRegEntryBase和EntryType之间没有任何关系，使用static_cast进行类型转换的意义是什么？

## MACRO
```c++
#define DMLC_REGISTRY_ENABLE(EntryType)                         \
  template<>                                                    \
  Registry<EntryType > *Registry<EntryType >::Get() {           \
    static Registry<EntryType > inst;                           \
    return &inst;                                               \
  }

#define DMLC_REGISTYR_REGISTER(EntryType, EntryTypeName, Name)  \
  static DMLC_ATTRIBUTE_UNUSED EntryType & __make_ ## EntryTypeName ## _ ## Name ## __ = \
      ::dmlc::Registry<EntryType>::Get()->__REGISTER__(#Name)   \

#define DMLC_REGISTRY_FILE_TAG(UniqueTag)                       \
    int __dmlc_registry_file_tag_ ## UniqueTag ## __() {return 0;}

#define DMLC_REGISTRY_LINK_TAG(UniqueTag)                       \
  int __dmlc_registry_file_tag_ ## UniqueTag ## __();           \
  static int DMLC_ATTRIBUTE_UNUSED  __reg_file_tag_ ## UniqueTag ## __ = \
      __dmlc_registry_file_tag_ ## UniqueTag ## __();
```
第一个宏上面已经说过了，第二个宏给定EntryType，EntryTypeName，Name，生成名为\_\_make\_EntryTypeName\_Name\_\_的EntryType引用，该引用绑定Registry<EntryType>中名为Name的EntryType变量。可能是因为注册并不代表接下来会用到，加上了DMLC\_ATTRIBUTE\_UNUSED (即\_\_attribute\_\_(unused))来通知编译器无需报warning。

最后两个宏是为了实现目标文件之间的强制连接，DMLC_REGISTRY_FILT_TAG是对应于UniqueTag的唯一函数定义，而DMLC_REGISTRY_LINK_TAG是对对应UniqueTag的函数的调用，由于该层调用关系，使得含有后者的目标文件一定会在静态链接阶段链入前者。
```c++
// main.cc

typedef void (*handler)(const char *protocol);
typedef map<const char *, handler> M;
M m;

void register_handler(const char *protocol, handler) {
  m[protocol] = handler;
}
int main(int argc, char *argv[])
{
  for (int i = 1; i < argc-1; i+= 2) {
    M::iterator it = m.find(argv[i]);
    if (it != m.end()) it.second(argv[i+1]);
  }
}

//http.c(part of libhttp.a)

class HttpHandler {
  HttpHandler() { register_handler("http", &handle_http); }
  static void handle_http(const char *) { /* whatever */ }
};
HttpHandler h; // registers itself with main!
```
注意到main本身不依赖http.c中的任何成员，因此在编译时，如果使用
```
g++ main.cc -lhttp
```
生成的文件中不会含有http的部分。另一种解决方法是编译时增加链接选项``--whole-archive``。
