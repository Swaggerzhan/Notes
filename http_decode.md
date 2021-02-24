# 使用C++解析HTTP协议


解析http协议请求之前，我们先来看看一个标准的http协议请求
```
POST /index.php　HTTP/1.1\r\n　　 
Host: ****\r\n
User-Agent: Mozilla/5.0 (Windows NT 5.1; rv:10.0.2) Gecko/20100101 Firefox/10.0.2\r\n　　
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,/;q=0.8\r\n
Accept-Language: zh-cn,zh;q=0.5\r\n
Accept-Encoding: gzip, deflate\r\n
Connection: keep-alive\r\n
Referer: http://localhost/\r\n
Content-Length：25\r\n
Content-Type：application/x-www-form-urlencoded\r\n
\r\n
```
其中第一行是属于请求行，其余都归为请求头，HTTP请求有一些比较明显的特征，比如说数据之前喜欢有空格(有的是\t)。行与行之间使用\r\n来结尾，在HTTP请求的结尾也会多出一个\r\n，我已都注明在其中。

有了这些特征，我们将用来解析HTTP请求的每一行
```C++
// 行的检查状态
enum LINE_STATUS{
    // 完整行
    LINE_OK = 0,
    // 出错
    LINE_BAD,
    // 数据还未完整
    LINE_OPEN,
};


/**
 * 解析行
 * buffer 数据开头
 * checked_index 目前已经检查过的数据的长度
 * read_index 目前所有数据的长度
 * 
 *
 **/
LINE_STATUS parse_line(char *buffer, int &checked_index, int &read_index) {
    char temp;
    /* 随着循环，我们将一个个检查数据的每一个字段，并且分析其中出现的行，并且将
    其中的\r\n替换为'\0'
    */
    for (; checked_index < read_index; ++checked_index){
        // 当前需要分析的字节
        temp = buffer[ checked_index ];
        // 如果一个字符串是\r，那么很有可以能这就是一个行
        if (temp == '\r'){
            // \r的下一个字符串还没有收到，需要等待客户端发送
            if ( (checked_index + 1) == read_index )
                return LINE_OPEN;
            // 获取到完整到行，将\r和\n的位置设置为\0
            if ( buffer[ checked_index + 1] == '\n'){
                buffer[checked_index ++] = '\0';
                buffer[checked_index ++] = '\0';
                // 这里可以return line ok 的原因是由于check_index和read_index是引用传参
                return LINE_OK;
            }
        // 这个地方可能是完整的行原因是由于上次\r收到后进行等待\n的到来
        }else if ( temp == '\n'){
            if ( (checked_index > 1) && buffer[ checked_index - 1] == '\r'){
                buffer[ checked_index - 1] = '\0';
                buffer[ checked_index ++ ] = '\0';
                return LINE_OK;
            }
            return LINE_BAD;
        }

    }
    return LINE_OPEN;
}
```
以上我们就可以用来解析每一个HTTP的字符，从而分析出每一个行，如果 __出错__ ，返回行`LINE_BAD`，如果 __正确__ ，返回`LINE_OK`，如果 __信息不完全__ ，则返回`LINE_OPEN`。但是这些还不够，后续我们还需要分析这每一个行的含义，但这些都是建立在`LINE_OK`上的，接下来，我们将写一段程序来 __解析HTTP的整个内容__ 。
```C++
// HTTP请求的响应码
enum HTTP_CODE {
    NO_REQUEST, 
    GET_REQUEST, // GET 请求
    BAD_REQUEST, // 请求错误
    FORBIDDEN_REQUEST, // 没有权限
    INTERNAL_ERROR, // 内部出错
    CLOSED_CONNECTION, // 客户端关闭链接
    // 其他请求响应码...
};

enum CHECK_STATE{
    //正在分析请求行，第一行
    CHECK_STATE_REQUESTLINE = 0,
    // 正在分析头部字段
    CHECK_STATE_HEADER,
};


/**
 * 用来整体解析HTTP内容，具体使用这个函数的方法将在下面放出
 * buffer 指向socket接收到的数据
 * checked_index代表已经检查过的字符串的长度
 * read_index表示当前buffer中存在的字符串长度(包括还没被检查的)
 * checkState表示当前检查状态
 * start_line用来区分每一行(也就是每一行的开头)
 *
 * */
HTTP_CODE parse_content(char *buffer, int &checked_index, CHECK_STATE &checkState, int &read_index, int &start_line) {
    // 开始，我们先将行设置为行正确以进行解析
    LINE_STATUS lineStatus = LINE_OK;
    HTTP_CODE retCode = NO_REQUEST;
    // 只要行正确，我们就一只解析，直到出错或者结束。。
    while ( (lineStatus = parse_line(buffer, checked_index, read_index)) == LINE_OK){
        // 第一次start_line是0，所以就是buffer的开头
        char *temp = buffer + start_line;
        // 之后的每次start_line都会在每一行的开头
        start_line = checked_index;

        switch ( checkState ){
            // 分析请求行
            case CHECK_STATE_REQUESTLINE: { 
                retCode = parse_request_line(temp, checkState);
                if (retCode == BAD_REQUEST)
                    return BAD_REQUEST;
                break;
            }
            // 分析请求头
            case CHECK_STATE_HEADER: {
                retCode = parse_headers( temp );
                if ( retCode == BAD_REQUEST )
                    return BAD_REQUEST;
                else if ( retCode == GET_REQUEST )
                    return GET_REQUEST;
                break;
            }

            default: {
                return INTERNAL_ERROR;
            }
        }

    }
    // 数据还不完整
    if ( lineStatus == LINE_OPEN)
        return NO_REQUEST;
    return BAD_REQUEST;
}
```

以上就是解析整个HTTP的主要函数，其中出现了2个函数，一个是parse_request_line，一个是parse_headers。
```C++
/**
 * 解析请求行
 * temp 请求行开始字符
 * checkState 状态
 **/
HTTP_CODE parse_request_line(char *temp, CHECK_STATE &checkState) {

    // strpbrk返回和s2中任何字符串匹配的位置
    char *url = strpbrk(temp, " \t");

    // url中没有任何空格和\t绝对有问题，返回错误
    if ( !url )
        return BAD_REQUEST;
    // 将空格设置为\0并且往后读取
    *url ++ = '\0';
    // 中间已经被截取了，所以这里可以直接拿到方法
    char *method = temp;

    if ( strcasecmp( method, "GET" ) == 0)
        printf("The Request method is GET\n");
    else if ( strcasecmp( method, "POST") == 0)
        printf("The Request method is POST\n");
    else if ( strcasecmp( method, "PUT") == 0)
        printf("The Request method is PUT\n");
    else
        return BAD_REQUEST;


    //分离url和版本信息
    url += strspn(url, " \t");
    char *version = strpbrk(url, " \t");
    if ( !version )
        return BAD_REQUEST;

    *version ++ = '\0';
    version += strspn(version, " \t");

    if ( strcasecmp( version, "HTTP/1.1") != 0)
        return BAD_REQUEST;

    if ( strncasecmp( url, "http://", 7) == 0){
        url += 7;
        url = strchr(url, '/');
    }

    if ( !url || url[0] != '/')
        return BAD_REQUEST;

    printf("The Request URL is: %s\n", url);
    checkState = CHECK_STATE_HEADER;
    return NO_REQUEST;

}

/**
 * 解析请求头
 * temp 字符串开头
 * 
 **/
HTTP_CODE parse_headers(char *temp) {

    // 这里返回GET，需要支持其他方法需要自行添加与修改
    if ( temp[0] == '\0')
        return GET_REQUEST;
    // 匹配Host和其他头等等....
    else if ( strncasecmp(temp, "Host:", 5) == 0){
        temp += 5;
        temp += strspn(temp, " \t");
        printf("host is: %s\n", temp);
    } else if ( strncasecmp(temp, "User-Agent:", 11) == 0){
        temp += 11;
        temp += strspn(temp, " \t");
        printf("user-agent is: %s\n", temp);
    }
    else{
        printf("not handle this header now!\n");
    }

    return NO_REQUEST;
}
```

全部的解析方式都在以上，下面是使用案例：
```C++


int main() {

    int sock = socket(AF_INET, SOCK_STREAM, 0);

    int reuse = 1;
    setsockopt(sock, SOL_SOCKET, SO_REUSEADDR, &reuse, sizeof(reuse));

    struct sockaddr_in local_addr;
    memset(&local_addr, 0, sizeof(local_addr));
    local_addr.sin_addr.s_addr = inet_addr("0.0.0.0");
    local_addr.sin_family = AF_INET;
    local_addr.sin_port = htons(9999);

    ::bind(sock, (sockaddr*)&local_addr, sizeof(local_addr));

    ::listen(sock, 5);
    // 我将方法封装在这个类中
    HTTP_Request* handler = new HTTP_Request;

    while (1){
        struct sockaddr_in client_addr;
        socklen_t client_addr_sz = sizeof(client_addr);
        int fd = accept(sock, (sockaddr*)&client_addr, &client_addr_sz);
        if ( fd < 0 ) {
            printf("error!\n");
            continue;
        }

        char buffer[BUFFER_SIZE];
        int data_read = 0;
        int read_index = 0;
        int checked_index = 0;
        int start_line = 0;

        CHECK_STATE checkState = CHECK_STATE_REQUESTLINE;
        while (1){
            data_read = recv(fd, buffer + read_index, BUFFER_SIZE - read_index, 0);
            if ( data_read == -1){
                printf("reading failed\n");
                break;
            }
            if ( data_read == 0){
                printf("remote closed\n");
                break;
            }
            read_index += data_read;
            HTTP_CODE result = handler->parse_content(buffer, checked_index,
                    checkState, read_index, start_line);

            // http请求尚未完整
            if ( result == NO_REQUEST )
                continue;

            else if ( result == GET_REQUEST ){
                send(fd, szret[0], strlen( szret[0]), 0);
                break;
            } else{
                send(fd, szret[1], strlen( szret[1]), 0);
                break;
            }
        }
        close(fd);

    }

    close(sock);
    return 0;


}
```

引用：Linux高性能服务器编程(游双)