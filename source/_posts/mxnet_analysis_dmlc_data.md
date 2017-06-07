---------------------------------------------------
title: MXNet源码分析(dmlc_core/include/dmlc/data.h)
---------------------------------------------------

data.h提供处理输入数据的通用接口类

-----------------
## class DataIter
-----------------
```c++
template<typename DType>
class DataIter {
 public:
  virtual ~DataIter(void) {}
  virtual void BeforeFirst(void) = 0；
  virtual bool Next(void) = 0;
  virtual const DType &Value(void) const = 0;
}
```
唯一的问题的是纯虚基类的dtor是否一定要是纯虚函数？


------------
## class Row
------------
```c++
template<typename IndexType>
class Row {
 public:
  ...
  size_t length;
  const IndexType* field;
  const IndexType* index;
  const real_t *value;
  ...
  inline IndexType get_index(size_t i) const {
    return index[i];
  }
  inline real_t get_value(size_t i) const {
    return value[i];
  }
  template<typename V>
  inline V SDot(const V *weight, size_t size) const {
    ...
  }
};
```
``Row``类应该是一个面向稀疏行的类型，``length``是非零元素的个数，``IndexType *index``是非零元素的index，``real_t *value``是非零元素的值，``SDot``方法提供了``Row``和一个dense向量做点积的实现。``IndexType *field``的作用未知？

## class RowBlock
```c++
template<typename IndexType>
struct RowBlock {
  ...
}
```
类似``class Row``，``class RowBlock``包含有一个系数矩阵的多个行，成员变量基本参照``Row``（单行变多行）。多了抽取其中一行以及抽取多行组成新的``RowBlock``的方法，采用复制指针的方法，不复制资源。注意到``Row``和``RowBlock``都不涉及资源的申请和释放，只提供基本的访问接口。


---------------------
## class RowBlockIter
---------------------
```c++
template<typename IndexType>
class RowBlockIter: public DataIter<RowBlock<IndexType> > {
 public:
  static RowBlockIter<IndexType> *Create(const char *uri,
                                         unsigned part_index,
                                         unsigned num_parts,
                                         const char *type);
  virtual size_t NumCol() const = 0;
};
```


---------------
## class Parser
---------------
```c++
template<typename IndexType>
class Parser: public DataIter<RowBlock<IndexType> > {
 public:
  static RowBlockIter<IndexType> *Create(const char *uri,
                                         unsigned part_index,
                                         unsigned num_parts,
                                         const char *type);
  virtual size_t BytesRead(void) const = 0;
  typedef Parser<IndexType>* (*Factory)(const std::string& path,
                                        const std::map<std::string, std::string>& args,
                                        unsigned part_index,
                                        unsigned num_parts);
};

template<typename IndexType>
struct ParserFactoryReg
    : public FunctionRegEntryBase<ParserFactoryReg<IndexType>,
                                  typename Parser<IndexType>::Factory> {};
```
最值得关注的是出现了``FunctionRegEntryBase``的用法，这里需要捋一下，``FuntionRegEntryBase<A, B>``本身是一个template，它的第一个模板参数A决定了其每一个成员函数的返回类型，尤其是对方法``self()``
```c++
 protected:
  inline EntryType &self() {
    return *(static_cast<EntryType*>(this));
  }
```
如果单看``self()``本身，是一个非常奇怪的行为，把``this``的类型转换成模板参数的类型。但是如果结合这里的使用方法来看，``ParserFactoryReg``继承自``FunctionRegEntryBase``，同时，``FunctionRegEntryBase``的模板参数又是``ParserFactoryReg``本身的类型，这就保证``self()``中的``static_cast``总是在两个有继承关系的类型之间进行。对上面这种设计模式进行抽象，大概就是
```c++
template<typename DeriveType>
class Base {
  DeriveType& f() {
    return *(static_cast<DeriveType*>(this));
  }
};

template<typename ParamType>
class Derive: public Base<Derive<ParamType> > {
  ...
};
```
``class Derive``继承自``class Base``，它调用``f()``总是会返回一个类型为它自己的类型``Derive<ParamType>``的引用，单从这里能看到的是这种模式为所有派生自Base的类提供了一个永远能够返回他们自身类型引用的方法。但是这种做法的好处和必要性，现在还没有看出来。


--------
## MACRO
--------
```c++
#define DMLC_REGISTER_DATA_PARSER(IndexType, TypeName, FactoryFunction) \
  DMLC_REGISTRY_REGISTER(::dmlc::ParserFactoryReg<IndexType>,           \
                         ParserFactoryReg ## _ ## IndexType, TypeName)  \
  .set_body(FactoryFunction)
```
``ParserFactoryReg<IndexType>``中，``self()``返回``ParserFactoryReg<IndexType>&``，``body``成员变量的类型为``Parser<IndexType>::Factory``。通过调用``DMLC_REGISTRY_REGISTER``,首先初始化了一个singleton ``Registry<ParserFactory<Index> >``，并且产生名为``__make_ParserFactoryReg_ ## IndexType ## _ ## TypeName ## _``，类为``ParaserFactory<IndexType>``的实例，且将该实例以``TypeName``为key加入``Registry<ParserFactory<Index> >``中。

对于每一种``IndexType``，生成与之对应的全局唯一的``Registry``，每个``TypeName``对应Registry中的一个``Entry``类元素。在模板参数相同的情况下，可能有着不同的生产方式（``FactoryFunction``），``TypeName``用来区分这些不同的生产方式，和``FactoryFunction``一一对应。

那么，register.h中``DMLC_REGISTRY_REGISTER<EntryType, EntryTypeName, Name>``实际上是在一个singleton的``Registry<EntryType>``中生成一个以``Name``为key的``Entry``。之所以相同的``EntryType``类有着多个不同``Name``，是因为它们的``Function``可能不同。以``PaserFunctionReg``为例，它是产生Parser的Factory的Registry，存在多种Factory方法
