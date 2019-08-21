这里主要介绍Gstreamer的几个基础元素。

# Elements
元素是Gstreamer中最重要的对象类型。Gstreamer使用管道设计模式，把多个元素串在一起，然后数据流会流经管道中的每个元素。可以通过这样的方式从文件中读取数据，然后解码数据，最后输出到你的声卡或其它设备或文件上。通过串连不同元素形成一个管道来实现媒体数据播放或捕捉功能。Gstreamer默认提供了大量的元素，为开发多媒体应用提供了多种的选择。如果已有的元素不能满足需求，你还可以开发自定义元素。