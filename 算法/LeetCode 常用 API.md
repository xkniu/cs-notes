# LeetCode 常用 API

## For-Each loop

`for (type var : iterable) statement`

## Array

常用操作：

```java
// 数组初始化
int[] a0 = new int[length];
int[] a1 = new int[]{1, 2, 3};

// 数组转换为 List
// 转换为不可变 list
List<Integer> list0 = Arrays.asList(a0);
// 构造新的可变 list
List<Integer> list1 = new ArrayList<>(Arrays.asList(a0));
// 创建 list 并把 array 添加进去
List<Integer> list2 = new ArrayList<>(a0.length);
Collections.addAll(list2, a0);

// 数组遍历
// index 遍历
for (int i = 0; i < a0.length; i++) {
    // ...
}
// for-each 遍历
for (int i : a0) {
    // ...
}
```

注意点：

- `a0.length` 是属性不是方法。

## List

常用操作：

```java
// 初始化
List<String> list0 = new ArrayList<>();
// 初始化并且指定容量
List<String> list1 = new ArrayList<>(10);
// 使用 list 来初始化
List<String> list2 = new ArrayList<>(Arrays.asList(new String[]{"hello", "world"}));
// 匿名内部类初始化并放入初始值
List<String> list3 = new ArrayList<>() {{ add("hello"); }};

// 遍历
// for-each 遍历
for (String str : list0) {
    // ...
}
// index 遍历
for (int i = 0; i < list0.size(); i++) {
    String e = list.get(i);
    // ...
}

// 在末尾添加元素
list0.add("test");
// 在指定 index 添加元素
list0.add(0, "test");
// 批量添加元素
list0.addAll(list1);
// 是否为空
boolean isEmpty = list0.isEmpty();

// 转换为 array
Object[] arr = list0.toArray();
String[] arr = list0.toArray(new String[0]);
```

注意点：

- 实现了 `RandomAccess` 接口的列表才适合使用 index 遍历。 `ArrayList` 适合使用 index 遍历，`LinkedList` 不适合。
- 如果知道最终会放多少条数据，初始化的时候应该指定容量避免扩容。

## 字符串

常用操作：

```java
// 字符串遍历
// 字符数组遍历
char[] cs = srt.toCharArray();
for (char c : cs) {
    // ...
}
// index 遍历
for (int i = 0; i < str.length(); i++) {
    char c = str.charAt(i);
}

// 构造字符串
StringBuilder sb = new StringBuilder();
sb.append("hello,");
sb.append("world");
String str = sb.toString();
```

注意点：

- `str.toCharArray()`：会额外复制一份字符数组，需要申请额外的空间。
- 字符串是不可变的，构造和修改字符串使用 `StringBuilder`。

## Map

常用操作：

```java
// 初始化
Map<Integer, Integer> map = new HashMap<>();

// 遍历
// entry 遍历
for (Map.Entry<Integer, Integer> entry : map.entrySet()) {
    // ...
}
// key 遍历
for (Integer key : m.keySet()) {
    // ...
}

// 键值对放入 map
map.put(1, 100);
// 根据键获取值
Integer value = map.get(1);
// 不存在时获取默认值
Integer value = map.getOrDefault(1, 0);
// 是否包含 key
boolean contains = map.containsKey(1);
// 是否包含 value
boolean containsValue = map.containsValue(1);
```

注意点：

- 需要按插入顺序遍历，可以使用 `LinkedHashMap`。
- 需要按 key 顺序遍历，可以使用 `TreeMap`。
