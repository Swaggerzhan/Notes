### enum

Rust中的enum，个人认为只要理解了“变体”(载体)，就基本上能够理解了，在C/C++中，我们会这么写enum：
```C
enum MyType {
    Type0 = 0,
    Type1 = 1,
    Type2 = 2,
    // ...
};
```
仿佛Type0的载体就是整数类型的0，Type1就是1，Rust扩展了这种定义，使得enum可以使用任意自定义内容作为载体，比如：

```Rust
enum XYZ {
    X,
    Y,
    Z,
}

enum MyType {
    Type0(i32),
    Type1(i32),
    Type2(u32),
    Type3(XYZ),
}
```

Type0的变体是i32，说白了就是Type0底层存储的内容是一个i32类型，同理Type3底层载体就是XYZ这个自定义的“enum”，需要注意的是，这里仅仅是“变体”，或者“载体”，不能和C/C++中的`static_cast<int>(MyType::Type0) == 0`这种相提并论，在Rust的Type中，MyType如果值是i32的1，那么他即可以是Type0，也能是Type1，取决于你想要以那种方式来解析它。

这种东西在Rust的Result中尤为重要，比如：
```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```
Result是一个enum的类型，存在2个generic类型，T和E(这点和C++ template<typename T, typename E>很类似)；
我们让Ok的载体为T，即返回值，Err的载体为E，即返回错误值，这样就能如此使用：

```rust
#[derive(Debug)]
enum IpAddr {
    Ipv4(u8, u8, u8, u8),
    Ipv6(String),
}

fn get_ip(want: i32) -> Result<IpAddr, String> {
    if want == 4 {
        // mock return ipv4
        Ok(IpAddr::Ipv4(192, 168, 1, 1))
    } else if want == 6 {
        Ok(IpAddr::Ipv6(String::from("::1")))
    } else {
        Err(String::from("ip version error!"))
    }
}

fn main() {

    let ip = get_ip(4);
    match ip {
        Ok(ipv4) => println!("we got ipv4 {:?}", ipv4),
        Err(err) => panic!("panic reason: {err}"),
    }

    let ip2 = get_ip(6);
    match ip2 {
        Ok(ipv6) => println!("we got ipv4 {:?}", ipv6),
        Err(err) => panic!("panic reason: {err}"),
    }

    let ip3 = get_ip(1);
    match ip3 {
        Ok(ipv4) => println!("we got ipv4 {:?}", ipv4),
        Err(err) => panic!("panic reason: {err}"),
    }
}
// output:
// we got ipv4 Ipv4(192, 168, 1, 1)
// we got ipv4 Ipv6("::1")
// thread 'main' panicked at src/main.rs:36:21:
// panic reason: ip version error!
```

如此，通过match(有点类似switch)，转化Ok的载体为我们想要的内容，比如Ipv4，Ipv6，String。


