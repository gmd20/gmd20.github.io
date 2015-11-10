1.
直接 filter 输入http 或者 http.time ，wireshark 每个http response    
包里面都有一个 “Time since request”, 显示这个http请求的响应时间的。   

2.
statistics  -》 IO Graph  -》  Y  Axis  -> Unit   "Advanced "  ,  
Calc “AVG” 可以选择 MIN MAX或者 AVG  后面 输入http.time    表示平均响应时间，    
style 选择 “FBar” 柱状图。   再点击  最前面的 “Graph 1”  显示第一个曲线图示。   
