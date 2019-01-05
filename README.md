# arduino 学习笔记

## pubsubclient 库 callback 的 payload 使用方法

使用 [knolleary](https://github.com/knolleary)/**[pubsubclient](https://github.com/knolleary/pubsubclient)** 发现，在[callback](https://github.com/knolleary/pubsubclient/blob/26ce89fa476da85399b736f885274d67676dacb8/examples/mqtt_publish_in_callback/mqtt_publish_in_callback.ino#L33) 返回值 `byte* payload`，C++基础薄弱，不知道如何处理，经群友帮助总算解决了。

由于返回的是一个 bate* 指针，这个指针指向了 payload 的内存地址，而这个指针地址上的字符串并没有归零，所以会导致 char 直接赋值错误。

### 1、 以 length 作为长度，历遍整个 payload

以下是我自己琢磨的第一版

```c
void callback(char* topic, byte* payload, unsigned int length) {
  Serial.println ("callback ...");
  String mqtt_message;
  for (int i=0;i<(int)length;i++){
    Serial.println((char)payload[i]);
    mqtt_message += (char)payload[i];
  }
  Serial.print("mqtt_message:");
  Serial.println(mqtt_message);
  if (mqtt_message.equals(String("reset"))) {
    Serial.println ("Reset ...");
    requestRestart = true;
  }
}
```

这个是参照范例写出来的，主要是我一开始没搞懂 String 对象和 char* 指针的关系，导致赋值一直错误，写成 `mqtt_message[i] = (char)payload[i];`，其实应该是 `mqtt_message += (char)payload[i];` 就好了。

### 2、先将 char* 转换成 String 然后再使用 toCharArray 方法以固定的 length 转会 char 做对比。

```c
void callback(char* topic, byte* payload, unsigned int length) {

  Serial.println ("Callback ....");
  Serial.print("Payload length:");
  Serial.println ((int)length);
 
  char PayloadChar[(int)length];
  char reset[ ] = "reset";
  String PayloadString;
  // Convert payload to String
  PayloadString = String((char*)payload);
  // Convert String to CharArray
  PayloadString.toCharArray(PayloadChar, (int)length);
  Serial.print ("PayloadChar:");
  Serial.println (PayloadChar);
  if (strcmp(reset,PayloadChar)){
    Serial.println ("reset");
  }else {
    Serial.println ("none");
  }
}
```

这个就十分曲折，其实在 `PayloadString = String((char*)payload);` 的时候，PayloadString 应该不是 length 长度的，只要没有遇到 0，他会一直赋值，遇到0位为止。

所以需要使用 `PayloadString.toCharArray(PayloadChar, (int)length);` 方法规定范围，我们只需要  reset ，payload 以外的东西我们都不要。

### 3、将 byte* 转换成 String 然后用 equals 方法对比

```c
void callback(char* topic, byte* payload, unsigned int length) {

  Serial.println ("Callback ....");
  Serial.print ("Payload lenth:");
  Serial.println ((int)length);

  String reset = String("reset");
  char PayloadChar[(int)length+1];
  memcpy(PayloadChar,(char*)payload,(int)length);
  PayloadChar[(int)length+1]=0;
  String PayloadString(PayloadChar);
  Serial.print ("PayloadString:");
  Serial.println (PayloadString);
  if (reset.equals(PayloadString)){
    Serial.println ("reset");
  }else {
    Serial.println ("none");
  }
}
```

这个我是看了上面第二个方法，感觉很绕，转来转去，既然都转成 String 了，为啥还要转会 char呢？所以就折腾出这个方法。这个是群友教导我的，使用 **memcpy** 方法来规定范围，将 byte* 缓冲区的内容，复制到新的 char 里面，并给新的缓冲区最后一位 + 0，也就是字符串终止符，那么在 String() 对象转换的时候就不会把其他垃圾也转换过来了。

总体而言，都是为了控制要 byte* payload  的缓冲区地址不要超过 length 范围。