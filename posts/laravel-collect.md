---
title: 'Laravel学习-Collect'
date: 2020-06-06 18:13:32
tags: [laravel,collect]
published: true
hideInList: false
feature: 
isTop: false
---

> 集合（Collection）Illuminate\Support\Collection 类了提供一个便捷的操作数组的封装。[Collection.php源码解读][1]

[TOC]

### 创建一个新的集合
 
```PHP
//collect 辅助函数会为指定的数组返回一个新的 Illuminate\Support\Collection 实例
//构造函数
public function __construct($items = [])
{
    //可以直接将多种类型转换为数组 Laravel Eloquent ORM 也以集合的形式返回数据
    $this->items = $this->getArrayableItems($items);
}
// 创建一个新的集合
$newCollection = collect([1, 2, 3, 4, 5]);
```

### 静态函数 times()
```PHP
静态 times 方法通过调用给定次数的回调函数来创建新集合：

public static function times($number, callable $callback = null)
{
    if ($number < 1) {
        return new static;
    }
    if (is_null($callback)) {// 回调函数为空，直接返回range()
        return new static(range(1, $number));
    }
    return (new static(range(1, $number)))->map($callback);//给定次数调用回调函数
}
// 基本用法
> Illuminate\Support\Collection::times(10, function ($number) {
    return $number * 9;
})all();
=> [9, 18, 27, 36, 45, 54, 63, 72, 81, 90]

// 回调函数为空，直接返回range()
> Illuminate\Support\Collection::times(2)->all();
=> [1,2,]
```
### 懶集合 LazyCollection
>LazyCollection 类利用了PHP的生成器来在保持低内存使用率的同时使用非常大的数据集 关键字（yield）。
```PHP
use App\LogEntry;
use Illuminate\Support\LazyCollection;

LazyCollection::make(function () {
    $handle = fopen('log.txt', 'r');
    while (($line = fgets($handle)) !== false) {
        yield $line;
    }
})->chunk(4)->map(function ($lines) {
    return LogEntry::fromLines($lines);
})->each(function (LogEntry $logEntry) {
    // Process the log entry...
});
```

### 基本数据处理
#### avg()
> avg($callback = null) 集合平均值，支持回调函数
```PHP
// 一般用法
>>> collect([1, 1, 2, 4])->avg();
=> 2

// 指定键
>>> collect([['foo' => 10], ['foo' => 10], ['foo' => 20], ['foo' => 40]])->avg('foo')
=> 20

// 回调函数
>>> collect([['foo' => 10], ['foo' => 20], ['foo' => 40]])->avg(function($val){return $val['foo']/10;});
=> 2.3333333333333
```

#### median()
> median($key = null) 集合中位数，可以指定键
```PHP
// 一般用法
>>> collect([1, 1, 2, 4])->median();
=> 1.5

// 指定键中位数
>>> collect([['foo' => 10], ['foo' => 10], ['foo' => 20], ['foo' => 40]])->median('foo');
=> 15
```

#### mode()
> mode($key = null) 集合众数 指定键的众数[一组数据中出现次数最多的数值，有可能是多个]
```PHP
// 一般用法
>>> collect([1, 1, 2, 4])->mode();
=> [1]

// 指定键众数
>>> collect([['foo' => 10], ['foo' => 10], ['foo' => 20], ['foo' => 20], ['foo' => 40]])->mode('foo');
=> [10,20]
```

#### collapse()
> 一个多个一维数组集合组装为单个一维数组集合
```PHP
// 多个一维数组的值 array_merege
>>> collect([[1, 2, 3], [4, 5, 6], [7, 8, 9]])->collapse()->all()
=> [1,2,3,4,5,6,7,8,9]
// 不适用多维数组
>>> collect([['foo' => 10], ['foo' => 10], ['foo' => 20], ['foo' => 20], ['foo' => 40]])->collapse()->all()
=> ["foo" => 40]
>>>
```

#### contains()
>contains($key, $operator = null, $value = null) 方法检查集合有否包含指定的元素
contains 方法用 “松散” 比较检查元素值，用 containsStrict 方法使用 “严格” 比较过滤。
```PHP
// 单个数值或者回調函數判断
>>> collect([1,2,3,4])->contains(1)
=> true
>>> collect([['foo' => 10],['foo' => 20]])->contains(function($val){ return $val['foo'] > 20 ;});
=> true

// 数组，键值对
>>> collect([['foo' => 10], ['foo' => 10], ['foo' => 40]])->contains(['foo' => 10])
=> true
>>> collect([['foo' => 10], ['foo' => 10], ['foo' => 40]])->contains('foo',10)
=> true
// 支持多種比較查詢 == <> > < !== 等等
>>> collect([['foo' => 10], ['foo' => 10], ['foo' => 40]])->contains('foo', '>', 10)
=> true
```

#### crossJoin()
> crossJoin(...$lists)  方法交叉连接指定数组或集合的值，返回所有可能排列的笛卡尔积

```PHP
// 数组循环迭代，每次每个集合中取一个 类似于 An1*M 
>>> collect([1,2])->crossJoin([3,4])->all()
=> [
     [1,3],
     [1,4],
     [2,3],
     [2,4],
   ]
>>>

```

#### diff()
>数组中array_diff()方法，本函数只检查了多维数组中的一维
```PHP
>>> collect([1,2])->diff([1,2,3,4])->all()
=> []
>>> collect([1,2,5])->diff([1,2,3,4])->all()
=> [2 => 5]
```
#### diffUsing()
#### diffAssoc()
#### diffAssocUsing()
#### diffKeys()
#### diffKeysUsing()
#### duplicates()
>duplicates($callback = null, $strict = false) 从集合中检索并返回重复的值
```PHP
// 一般用法
>>> collect(['a', 'b', 'a', 'c', 'b'])->duplicates()->all()
=> [2 => "a", 4 => "b"]
// 如果集合包含数组或对象，则可以需要检查重复值的属性的键
>>> collect([['foo' => 10], ['foo' => 10], ['foo' => 40]])->duplicates('foo')->all();
=> [1 => 10]
```
#### duplicatesStrict()
#### except()
> except($keys) 方法返回集合中除了指定键之外的所有集合项：
```PHP
// 参数可传单个字段，也可以传枚举类型的数组
>>> collect(['foo' => 10, 'foo2' => 10, 'foo' => 40])->except('foo')->all();
=> ["foo2" => 10]
>>> collect(['foo' => 10, 'foo2' => 10, 'foo3' => 40])->except(['foo','foo2'])->all();
=> ["foo3" => 10]
```

#### filter()
> 用给定的回调函数过滤集合，只保留那些通过指定条件测试的集合项;如果没有提供回调函数，集合中所有返回 false 的元素都会被移除
```PHP
// 指定回调函数，过滤集合
>>> collect([['foo' => 10], ['foo' => 10], ['foo' => 40]])->filter(function($val){ return $val['foo'] > 20;})->all();
=> [
     2 => [
       "foo" => 40,
     ],
   ]
// 没有回调函数，直接过滤false array_filter()
>>> collect([1, 2, 3, null, false, '', 0, []])->filter()->all()
=> [1, 2, 3]
```

#### first()
> first(callable $callback = null, $default = null) 从集合中返回符合条件的第一个值，支持回调函数；可设置默认值$default
```PHP
>>> collect()->first();
=> null
>>> collect([1, 2, 3, 4])->first();
=> 1
>>> collect([1, 2, 3, 4])->first(function($val){ return $val > 30;});
=> null
>>> collect([1, 2, 3, 4])->first(function($val){ return $val > 3;});
=> 4
>>> collect([1, 2, 3, 4])->first(function($val){ return $val > 5;}, 5);
=> 5
```

#### flatten()
> flatten($depth = INF) flatten($depth = INF) 将多维集合转换为一维集合，其中 $depth 为转换深度
```PHP
>>> collect([['foo' => 10], ['foo' => 10], ['foo' => 40]])->flatten()->all();
=> [10,10,40]
>>> collect([1=>'a', 2=>['b'=>['c' => 'd']]])->flatten(2)->all()
=> ["a","d",]
```

#### flip()
#### forget()
> forget($keys) 通过指定的键来移除集合中对应的内容
```PHP
>>> collect(['foo' => 10, 'foo2' => 10, 'foo3' => 40])->forget(['foo','foo2'])->all();
=> ["foo3" => 40]
```
#### get()
> get($key, $default = null) 方法返回指定键的集合项，如果该键在集合中不存在，则返回 null；可传递默认参数default，该默认参数可为回调函数
```PHP
>>> collect(['a' => 1])->get('a')
=> 1
>>> collect(['a' => 1])->get('b')
=> null
>>> collect(['a' => 1])->get('b', 11)
=> 11
>>> collect(['a' => 1])->get('b', function(){return 111;})
=> 111
>>>
```

#### groupBy()
#### keyBy()
> 方法以指定的键作为新集合的键。如果多个集合项具有相同的键，则只有最后一个集合项会显示在新集合中;支持对调函数 
```PHP
// 一般使用，直接传递某个键
>>> collect([[ 'a'=>1,'b'=>2],['a'=>3,'b'=>4]])->keyBy('a')->all()
=> [
     1 => ["a" => 1,"b" => 2,],
     3 => ["a" => 3,"b" => 4,],
   ]
// 回调函数返回的值会作为该集合的键   
>>> collect([[ 'a'=>1,'b'=>2],['a'=>3,'b'=>4]])->keyBy(function($item){return $item['a']+10;})->all()
=> [
     11 => ["a" => 1,"b" => 2,],
     13 => ["a" => 3,"b" => 4,],
   ]
```

#### has()
> has($key) 判断集合中是否存在指定键,支持传入多个键 array_key_exists底层方法
```PHP
>>> collect([ 'a'=>1,'b'=>2, 'c'=>3])->has('a')
=> true
>>> collect([ 'a'=>1,'b'=>2, 'c'=>3])->has(['a','b'])
=> true
>>> collect([ 'a'=>1,'b'=>2, 'c'=>3])->has(['a','b','d'])
=> false
```

#### implode
> implode($value, $glue = null) 用于合并集合项
```PHP
// 集合中包含简单的字符串或数值
>>> collect([ 'a'=>1,'b'=>2, 'c'=>3])->implode('*')
=> "1*2*3"
// 集合包含数组或对象
>>> collect([[ 'a'=>1,'b'=>2],['a'=>3,'b'=>4]])->implode('a', '**')
=> "1**3"
```

#### intersect()
#### intersectByKeys()
#### isEmpty()
#### join()
>join($glue, $finalGlue = '') 将集合中的值用字符串连接
```PHP
>>> collect(['a', 'b', 'c'])->join(', ');
=> 'a, b, c'
>>> collect(['a', 'b', 'c'])->join(', ', ', and ');
=> 'a, b, and c'
>>> collect(['a', 'b'])->join(', ', ' and ');
=> 'a and b'
>>> collect(['a'])->join(', ', ' and '); 
=> 'a'
>>> collect([])->join(', ', ' and '); 
=> ''
```

#### keys()
#### last()
>last() 返回集合中通过指定条件测试的最后一个元素
```PHP
// 一般调用，直接返回最后有一个元素
>>> collect([ 'a'=>1,'b'=>2, 'c'=>3])->last();
=> 3
// 回调函数，返回符合条件的最后一个元素
>>> collect([ 'a'=>1,'b'=>2, 'c'=>3])->last(function($val){return $val<2;});
=> 1
```

#### pluck()
> pluck($value, $key = null) 方法可以获取集合中指定键对应的所有值
```PHP
// 指定键对应的所有值
>>> collect([[ 'a'=>1,'b'=>2],['a'=>3,'b'=>4]])->pluck('a')->all();
=> [1,3]
// 指定生成集合的键
>>> collect([[ 'a'=>1,'b'=>2],['a'=>3,'b'=>4]])->pluck('a', 'b')->all();
=> [2 => 1, 4 => 3]
// 如果存在重复的键，则最后一个匹配元素将被插入到弹出的集合中
>>> collect([['a'=>1,'b'=>2],['a'=>3,'b'=>2]])->pluck('a', 'b')->all();
=> [2 => 3]
```

#### map()
> map(callable $callback) 遍历集合并将每一个值传入给定的回调函数, 生成被修改过集合项的新集合
```PHP
>>> collect([[ 'a'=>1,'b'=>2],['a'=>3,'b'=>2]])->map(function($item, $key){ return $item['a'] * 2;})->all();
=> [2,6]
```

#### mapToDictionary()
#### mapWithKeys() 
>遍历集合并将每个值传入给定的回调函数。将返回一个包含单个键 / 值对的关联数组
```PHP
>>> collect([[ 'a'=>1,'b'=>2],['a'=>3,'b'=>4]])->mapWithKeys(function($item){return [$item['a'] => $item['b']];})->all()
=> [
     1 => 2,
     3 => 4,
   ]
```

#### merge() 
>方法将合并指定的数组或集合到原集合，如果给定的集合项的字符串键与原集合中的字符串键相匹配，则指定集合项的值将覆盖原集合的值 array_merge
```PHP
// 一般使用
>>> collect([ 'a'=>1,'b'=>2])->merge(['c' => 5])->all();
=> [
     "a" => 1,
     "b" => 2,
     "c" => 5,
   ]
// 有覆盖
>>> collect([ 'a'=>1,'b'=>2])->merge(['a' => 5])->all();
=> [
     "a" => 5,
     "b" => 2,
   ]
```
#### mergeRecursive()
> 以递归的形式合并给定的数组或集合到原集合中 array_merge_recursive
```PHP
// 同一个键值的合并
>>> collect(['a'=>1,'b'=>2])->mergeRecursive(['a' => 5])->all();
=> ["a" => [1,5,"b" => 2]
>>> collect([ 'a'=>['c'=>1],'b'=>2])->mergeRecursive(['a' => 5])->all();
=> ["a" => ["c" => 1,0 => 5,],"b" => 2,]
```

#### combine()
> 将一个集合的值作为键，与另一个数组或集合的值进行结合 array_combine
```PHP
// 一般使用
>>> collect(['a', 'b'])->combine([1,2])->all()
=> [
     "a" => 1,
     "b" => 2,
   ]
// 只能传递相等数量的集合
>>> collect(['a', 'b'])->combine([1,2,3])->all()
PHP Warning:  array_combine(): Both parameters should have an equal number of elements in D:/dev/study/laravel/vendor/laravel/framework/src/Illuminate/Support/Collection.php on line 777
=> [
     false,
   ]
```

#### union()
> 方法将给定数组添加到集合中。如果给定的数组含有与原集合一样的键，则首选原始集合的值,这点和merge相反
```PHP
>>> collect(['a'=>1, 'b'=>2])->union(['a'=>3,'d'=>4])->all()
=> [
     "a" => 1,
     "b" => 2,
     "d" => 4,
   ]
>>> collect(['a'=>1, 'b'=>2])->merge(['a'=>3,'d'=>4])->all()
=> [
     "a" => 3,
     "b" => 2,
     "d" => 4,
   ]

```

#### nth()

#### only()
> 返回集合中所有指定键的集合项
```PHP
>>> collect(['a'=>1, 'b'=>2])->only('a')->all()
=> [
     "a" => 1,
   ]
```

#### pop()
#### prepend()
> prepend($value, $key = null) 将指定的值添加的集合的开头
```PHP
// 添加指定值到集合的开头
>>> collect(['a'=>1,'b'=>2])->prepend('c')->all()
=> [
     0 => "c",
     "a" => 1,
     "b" => 2,
   ]
// 指定新增集合项的键   
>>> collect(['a'=>1,'b'=>2])->prepend('c',1)->all()
=> [
     1 => "c",
     "a" => 1,
     "b" => 2,
   ]
```
#### push()
> push($value) 添加一个值于集合末尾,只能插入一个值
```PHP
>>> collect(['a','b'])->push('c','d')->all()
=> ["a", "b", "c"]
>>> collect([1, 2, 3])->push(4, 5)->all();
=> [1, 2, 3, 4]
```
#### concat()
> 在集合的末端附加指定的数组或者集合
```PHP
>>> collect([1, 2, 3])->concat([4, 5])->all();
=> [1, 2, 3, 4, 5]
```

#### pull()
> pull($key, $default = null) 指定键对应的值从集合中移除并返回,default为默认值
```PHP
>>> collect(['a'=>1,'b'=>2])->pull('a',0)
=> 1
>>> collect(['a'=>1,'b'=>2])->pull('c',3)
=> 3
```

#### put()
> put($key, $value) 设定集合内指定键的值，如果不存在，则新增到末尾
```PHP
>>> collect(['a'=>1,'b'=>2])->put('c',3)->all()
=> [
     "a" => 1,
     "b" => 2,
     "c" => 3,
   ]
>>> collect(['a'=>1,'b'=>2])->put('a',3)->all()
=> [
     "a" => 3,
     "b" => 2,
   ]
```

#### random
> random($number = null) 集合中返回一个随机项,$number不能大于集合的数量，否则报错
```PHP
// 默认返回一个
>>> collect(['a'=>1,'b'=>2])->random()
=> 2
>>> collect(['a'=>1,'b'=>2])->random(2)->all()
=> [
     1,
     2,
   ]
// 大于集合数量，报错   
>>> collect(['a'=>1,'b'=>2])->random(3)->all()
InvalidArgumentException with message 'You requested 3 items, but there are only 2 items available.'
```

#### reduce
> reduce(callable $callback, $initial = null) 将每次迭代的结果传递给下一次迭代，直到遍历完集合
```PHP
// 循环迭代 返回res
>>> collect([1, 2, 3])->reduce(function($res,$val){ return $res+$val;});
=> 6
// res初始值
>>> collect([1, 2, 3])->reduce(function($res,$val){ return $res+$val;},10);
=> 16
```

#### replace
> replace 方法类似于 merge，不仅可以覆盖匹配到的相同字符串键的集合项，而且也可以覆盖数字键的集合
```PHP
>>> collect(['a'=>1,'b'=>2])->replace(['b'=>10])->all();
=> [
     "a" => 1,
     "b" => 10,
   ]
>>> collect([1, 2, 3])->replace([1=>10])->all();
=> [1, 10, 3]
```

#### replaceRecursive()

#### reverse()

#### search()
>search($value, $strict = false) 集合的搜寻方法，搜索给定的值,返回键值
```PHP
// 搜寻指定的值
>>> collect(['a'=>1,'b'=>2])->search('a');
=> false
>>> collect(['a'=>1,'b'=>2])->search('1');
=> "a"
// 指定严格模式
>>> collect(['a'=>1,'b'=>2])->search('1',true);
=> false
// 指定回调函数
>>> collect(['a'=>1,'b'=>2])->search(function($item,$key){return $item > 1;});
=> "b"
```

#### shift()
#### shuffle()
#### skip()
#### slice()
> slice($offset, $length = null) 返回集合中给定索引开始后面的部分： array_slice()
```PHP
>>> collect([1, 2, 3, 4, 5, 6, 7])->slice(3)->all()
=> [
     3 => 4,
     4 => 5,
     5 => 6,
     6 => 7,
   ]
>>> collect([1, 2, 3, 4, 5, 6, 7])->slice(2,2)->all()
=> [
     2 => 3,
     3 => 4,
   ]
```

#### split()
#### chunk()
> chunk($size) 集合分割成多个指定大小的较小集合
#### sort()
#### sortDesc()
#### sortBy()
#### sortByDesc()
#### sortKeys()
#### sortKeysDesc()
#### splice()
#### take()
#### takeUntil()
#### takeWhile()
#### transform()
> transform(callable $callback) 方法会遍历整个集合，并对集合中的每项都会调用其回调函数,和map原理一致，但是该方法会改变原有集合
```PHP
// map不会改变原集合的值
>>> $a = collect([1,2,3,4,5]);
>>> $a->map(function($item,$key){return $item*2;})
>>> $a->all();
=> [1,2,3,4,5,]
// transform改变原集合
>>> $a->transform(function($item,$key){return $item*2;})
>>> $a->all();
=> [2,4,6,8,10,]
>>>
```
#### values()
#### zip()
#### pad()
#### getIterator()
#### count()
#### add()
#### toBase()
> 基于当前集合返回一个新的集合
#### offsetExists()
#### offsetGet()
#### offsetSet()
#### offsetUnset()





  [1]: https://github.com/xiaoxie110/laravel/blob/master/vendor/laravel/framework/src/Illuminate/Support/Collection.php