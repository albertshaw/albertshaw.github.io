title: 如何通过JS获取本地JPG图片的高宽
date: 2014-12-24 21:13:18
tags: [FileReader, jpg dimension, javascript, height, DataView, width, jpg]
---
做个绘制SVG描边动画的Web App这个脑洞已经打开了很久了，但是一个月工时顶平常两个月这种状态持续太久，所以一直没来得及动手。现在终于可以开始着手做了。

既然是描边动画，那么SVG的viewBox参数最好和背景图片的高宽是一致的，这样等比缩放时，SVG的path不至于乱掉。作为一个后台开发，玩前端总是容易遇到各种问题，那么第一个问题来了，如何通过JS获取本地JPG图片的高宽呢？

因为input file拿不到真实的本地文件地址，所以期望input file拿到图片本地地址传给img src然后看高宽是不行的，然后在查W3C File API时看到了FileReader。

<!--more-->

FileReader是W3C File API中定义的一个接口，[接口完整定义看这里](http://www.w3.org/TR/FileAPI/#dfn-filereader)，只要不是古董浏览器就都有它的实现。

```javascript
	  // async read methods
	  void readAsArrayBuffer(Blob blob);
	  void readAsText(Blob blob, optional DOMString label);
	  void readAsDataURL(Blob blob);
```
第一眼就看到了readAsDataURL方法，将本地图片读成dataURL再放入img然后读高宽很简单嘛。于是方法一就出来了：

##### 方法一

```javascript
  var fileChooser = document.getElementById('filechooser');
  fileChooser.onchange = function(evt) {
    var reader = new FileReader();
    reader.onload = function(e) {
      var img = document.createElement('img');
      img.src = e.target.result;
      console.log(img.width);
      console.log(img.height);
    };
    reader.readAsDataURL(evt.target.files[0]);
  }
```

很可惜的是这样在FireFox中不兼容，读出来的高宽为0，需要将img追加进DOM才能读出高宽来。shemegui...完全不能接受啊...

于是看到了readAsArrayBuffer方法，能把图片读成binary data buffer，脑洞开一开，完全可以自己解析jpg图片格式的嘛。

jpeg格式标准是这样的,二进制数据由两个字节的标记位的来分块，前一个字节是0xFF，后一个字节标记不同分块如下：
```
  SOI 0xD8　 图像开始Start of Image
　APP0 0xE0　 JFIF应用数据块 Application 0
　APPn 0xE1 - 0xEF　 其他的应用数据块 Application(n, 1～15)
　DQT 0xDB　 量化表 Difine Quantization Table
　SOF0 0xC0　 帧开始 Start of Frame
　DHT 0xC4　 霍夫曼表 Difine Huffman Table
　SOS 0xDA　 扫描线开始 Start of Scan
　EOI 0xD9　 图像结束 End of Image
```
SOF的第6和第7字节标记图片高度，第8第9字节标记图片宽度。这样定位FileReader读出来的二进制数据数组就能拿到高宽了。

readAsArrayBuffer读出来是个ArrayBuffer对象，该对象定义可参考[khronos组织定义的Typed Array](https://www.khronos.org/registry/typedarray/specs/latest/#5)或[ES6 draft中的定义](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer)

而要真正访问这个数据则需要用到DataView，可参考[khronos这里](https://www.khronos.org/registry/typedarray/specs/latest/#8)或[ES6 draft这里](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DataView)

ES5中似乎没看到这两个对象的定义，所以，放弃古董浏览器吧。

##### 方法二
```javascript
  var fileChooser = document.getElementById('filechooser');
  fileChooser.onchange = function(evt) {
    var reader = new FileReader();
    reader.onload = function(e) {
      console.log(getImgDimension(new DataView(e.target.result)));
    };
    reader.readAsArrayBuffer(evt.target.files[0]);
  }

  function getImgDimension(dataView) {
    if ((dataView.getUint8(0) != 0xFF) || (dataView.getUint8(1) != 0xD8)) {
      return false; //SOI标记位不对，不是jpg图片
    }

    var offset = 2, length = dataView.byteLength, marker;
    while (offset < length) {
      if (dataView.getUint8(offset) != 0xFF) {
        return false; //不是图片标记位
      }

      marker = dataView.getUint8(offset + 1);
      console.log(marker.toString(16));
      if (marker == 0xC0) {
        //标记位0xC0表示图片帧SOF开始
        return {
          height : dataView.getUint16(offset + 5),
          width : dataView.getUint16(offset + 7)
        };
      } else {
        // 非图片帧则跳过该分块，每个分块的长度由该分块第3第4两个字节标识
        offset += 2 + dataView.getUint16(offset + 2);
      }
    }
  }
```

能拿到本地文件的二进制数据，感觉浏览器能做的事情又多了很多啊。
