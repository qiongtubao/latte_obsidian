
---
#rust-map-entry
在 Rust 中，`std::collections::HashMap` 中的 `entry` 方法可以用于获取一个指定键的 Entry 对象，Entry 对象可以用于插入、更新或删除该键对应的值。  
  
`entry` 方法接受一个键作为参数，返回一个 `std::collections::hash_map::Entry` 枚举类型的值，该枚举类型有两个变体：`Occupied` 和 `Vacant`。如果该键已经存在于 HashMap 中，则返回 `Occupied` 变体，否则返回 `Vacant` 变体。  
  
`Occupied` 变体包含一个对应值的可变引用，可以用于更新或删除该键对应的值。`Vacant` 变体包含一个插入该键值对的方法 `insert`，可以用于插入新的键值对。  
  
下面是一个使用 `entry` 方法的例子：  
  
```rust  
use std::collections::HashMap;  
  
let mut map = HashMap::new();  
map.insert("hello", 1);  
  
match map.entry("hello") {  
	std::collections::hash_map::Entry::Occupied(mut entry) => {  
		*entry.get_mut() += 1;  
	}  
	std::collections::hash_map::Entry::Vacant(entry) => {  
		entry.insert(1);  
	}  
}  
  
assert_eq!(map.get("hello"), Some(&2));  
```  
  
在上面的例子中，我们创建了一个空的 HashMap，并插入了一个键值对 "hello" => 1。然后，我们使用 `entry` 方法获取键为 "hello" 的 Entry 对象，并根据 Entry 的变体进行不同的操作。由于该键已经存在于 HashMap 中，我们获取到了 `Occupied` 变体，将对应值加 1。最后，我们断言该键对应的值为 2。  
  
总之，`entry` 方法是一个非常有用的 HashMap 方法，可以帮助我们更方便地插入、更新或删除键值对。

---

#rust-map-Entry-or_insert_with_key

在 Rust 中，`or_insert_with_key` 是 `std::collections::HashMap` 中的一个方法，用于插入一个键值对到 HashMap 中。如果该键已经存在于 HashMap 中，则返回对应的值的可变引用；否则，插入一个新的键值对，并返回该值的可变引用。  
  
`or_insert_with_key` 方法接受一个闭包作为参数，该闭包用于生成新的值。闭包的参数是键的引用，返回值是新值的引用。如果该键已经存在于 HashMap 中，则闭包不会被调用，直接返回对应的值的可变引用；否则，闭包会被调用，生成新的值，并将其插入到 HashMap 中。  
  
下面是一个使用 `or_insert_with_key` 方法的例子：  
  
```rust  
use std::collections::HashMap;  
  
let mut map = HashMap::new();  
let key = "hello";  
let value = map.entry(key).or_insert_with_key(|k| k.len());  
println!("{}", value); // 输出 5  
```  
  
在上面的例子中，我们创建了一个空的 HashMap，并将字符串 "hello" 作为键插入到 HashMap 中。由于该键不存在于 HashMap 中，闭包被调用，生成一个新的值 5，并将其插入到 HashMap 中。最后，我们通过可变引用 `value` 获取到了新值，并将其打印出来。  
  
总之，`or_insert_with_key` 方法是一个非常有用的 HashMap 方法，可以帮助我们更方便地插入和获取键值对。

---

