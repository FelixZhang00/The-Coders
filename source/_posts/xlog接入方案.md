# xlog接入方案

title:  xlog接入方案
tag: c++ 开源代码
---

[mars](https://github.com/Tencent/mars) 是微信最近开源的终端基础组件，是一个使用 C++ 编写的基础组件。
xlog是其中一个可单独使用的高性能日志模块。
本文将简单介绍下xlog的特点，并给出一个自定义的输出到文件的策略。

---
## xlog的特点

使用流式压缩方式对单行日志进行压缩，压缩加密后写进作为 log 中间 buffer的 mmap 中，当 mmap 中的数据到达一定大小后再写进磁盘文件中。
![](http://ww4.sinaimg.cn/large/8b331ee1gw1fbmxsvozhbj20jg08iglr.jpg)
![](http://ww3.sinaimg.cn/large/8b331ee1gw1fbmxtm3kajj20kp04caa7.jpg)

输出到文件的主要实现是在 Appender 模块也是可插拔的，如果对默认的策略不满意可以自己实现一套。
![](http://ww3.sinaimg.cn/large/8b331ee1gw1fblriu5wrqj20ci0630sv.jpg)

xlog还存在一些其他策略：

* 每次启动的时候会清理日志，防止占用太多用户磁盘空间
* 为了防止 sdcard 被拔掉导致写不了日志，支持设置缓存目录，当 sdcard 插上时会把缓存目录里的日志写入到 sdcard 上

从下面的log2file流程图中可以看出xlog是如何利用cahce目录解决插拔sdcard的问题的。
`log2file流程图`
![](http://ww4.sinaimg.cn/large/8b331ee1gw1fbmlojbj7dj210i19648b.jpg)


### 上一次没写完的日志，如何重新写到日志中 
在日志模块初始化会执行如下的代码，sg_log_buff为与mmap文件映射的逻辑内存，这里会主动调用Flush，把mmap文件中的数据（即上一次没写到日志文件中的日志）fluse到buffer中，并调用__log2file写到日志文件中。

```cpp
char mmap_file_path[512] = {0};
snprintf(mmap_file_path, sizeof(mmap_file_path), "%s/%s.mmap2", sg_cache_logdir.empty()?_dir:sg_cache_logdir.c_str(), _nameprefix);

    bool use_mmap = false;
    if (OpenMmapFile(mmap_file_path, kBufferBlockLength, sg_mmmap_file))  {
        sg_log_buff = new LogBuffer(sg_mmmap_file.data(), kBufferBlockLength, true);
        use_mmap = true;
    }
    AutoBuffer buffer;
    sg_log_buff->Flush(buffer); // 把mmap文件中日志信息flush到内存中，下面调用__log2file写到文件中。
    if (buffer.Ptr()) {
        __writetips2file("~~~~~ begin of mmap ~~~~~\n");
        __log2file(buffer.Ptr(), buffer.Length());
        __writetips2file("~~~~~ end of mmap ~~~~~%s\n", mark_info);
    }
```

---

## 修改xlog默认的输出到文件的策略

### xlog默认的策略
每次启动时会删除过期文件，只保留十天内的日志文件(该值定义在appender.cc中的 kMaxLogAliveTime )，所以给 Xlog 的目录请使用单独目录，防止误删其他文件。目前不会根据文件大小进行清理。
```cpp
static void __del_timeout_file(const std::string& _log_path) {
    time_t now_time = time(NULL);
    
    boost::filesystem::path path(_log_path);
    
    if (boost::filesystem::exists(path) && boost::filesystem::is_directory(path)){
        boost::filesystem::directory_iterator end_iter;
        for (boost::filesystem::directory_iterator iter(path); iter != end_iter; ++iter) {
            time_t fileModifyTime = boost::filesystem::last_write_time(iter->path());
            
            if (now_time > fileModifyTime && now_time - fileModifyTime > kMaxLogAliveTime) {
                if (boost::filesystem::is_regular_file(iter->status())) {
                    boost::filesystem::remove(iter->path());
                }
                else if (boost::filesystem::is_directory(iter->status())) {
                    __del_files(iter->path().string());
                }
            }
        }
    }
}
```
日志文件是按天命名的，每天产生一个日志文件。
在`__openlogfile`、`__log2file`中会调用。
```cpp
static void __make_logfilename(const timeval& _tv, const std::string& _logdir, const char* _prefix, const std::string& _fileext, char* _filepath, unsigned int _len) {
    time_t sec = _tv.tv_sec;
    tm tcur = *localtime((const time_t*)&sec);

    std::string logfilepath = _logdir;
    logfilepath += "/";
    logfilepath += _prefix;
    char temp [64] = {0};
    snprintf(temp, 64, "_%d%02d%02d", 1900 + tcur.tm_year, 1 + tcur.tm_mon, tcur.tm_mday);
    logfilepath += temp;
    logfilepath += ".";
    logfilepath += _fileext;
    strncpy(_filepath, logfilepath.c_str(), _len - 1);
    _filepath[_len - 1] = '\0';
}
```
比如在2017年01月10号的日志会写到LOG_20170110.log文件中，到2017年01月11号时，日志写到LOG_20170111.log，依次类推。过期的日志文件会在日志模块初始化时被清理掉。

### 改写输出到文件的策略
xlog输出的文件的逻辑都在`appender.cc`中实现，可以修改这里的代码实现一套自己的策略。
比如想要控制日志文件的大小，即Rotating file based on size的策略。
这里的实现方案并不支持cachedir。
```cpp
static bool __writefile(const void* _data, size_t _len) {
    if (NULL == sg_logfile) {
        assert(false);
        return false;
    }

    long before_len = ftell(sg_logfile);
    if (before_len < 0) return false;

    // 如果发生了__roate，需要reopen sg_logfile
    if(before_len+_len > sg_max_size){
        if(!__roate()){
            return false;
        }
    }

    if (1 != fwrite(_data, _len, 1, sg_logfile)) {
        int err = ferror(sg_logfile);

        __writetips2console("write file error:%d", err);


        ftruncate(fileno(sg_logfile), before_len);
        fseek(sg_logfile, 0, SEEK_END);

        char err_log[256] = {0};
        snprintf(err_log, sizeof(err_log), "\nwrite file error:%d\n", err);

        char tmp[256] = {0};
        size_t len = sizeof(tmp);
        LogBuffer::Write(err_log, strnlen(err_log, sizeof(err_log)), tmp, len);

        fwrite(tmp, len, 1, sg_logfile);

        return false;
    }

    return true;
}

template <typename T>
std::string NumberToString ( T Number )
{
     std::ostringstream ss;
     ss << Number;
     return ss.str();
}

static std::string __calc_filename(const std::string& _logdir, const char* _prefix, const std::string& _fileext,unsigned int index){
    std::string logfilepath = _logdir;
    logfilepath += "/";
    logfilepath += _prefix;
    if(index){
        logfilepath += ".";
        logfilepath += NumberToString(index);
    }
    logfilepath += ".";
    logfilepath += _fileext;
    return logfilepath;
}


    // Rotate files:
    // log.txt -> log.1.txt
    // log.1.txt -> log2.txt
    // log.2.txt -> log3.txt
    // log.3.txt -> delete
static bool __roate(){
    if(sg_logfilename.empty() || sg_logdir.empty() || sg_logfileprefix.empty()){
        return false;
    }

    fclose(sg_logfile);
    sg_logfile = NULL;

    for(auto i = sg_max_files; i > 0; --i){
        std::string src = __calc_filename(sg_logdir,sg_logfileprefix.c_str(),LOG_EXT,i-1);
        std::string target = __calc_filename(sg_logdir,sg_logfileprefix.c_str(),LOG_EXT,i);
        boost::filesystem::path src_path(src);
        boost::filesystem::path target_path(target);

        if(boost::filesystem::exists(target_path)){
            boost::filesystem::remove(target_path);
        }
        if(boost::filesystem::exists(src_path)){
            boost::filesystem::rename(src_path,target_path);
        }

    }

    //reopen
    sg_logfile = fopen(sg_logfilename.c_str(), "wb");
    if (NULL == sg_logfile) {
        __writetips2console("open file error:%d %s, path:%s", errno, strerror(errno), sg_logfilename.c_str());
        return false;
    }

    return true;
}

static bool __openlogfile(const std::string& _log_dir){
    if (sg_logdir.empty()) return false;

    // 如果sg_logfile已经打开了，则直接返回。
    if (NULL != sg_logfile) {
        return true;
    }

    sg_current_dir = _log_dir;
    sg_logfilename = __calc_filename(_log_dir, sg_logfileprefix.c_str(), LOG_EXT,0);

    sg_logfile = fopen(sg_logfilename.c_str(), "ab");

    if (NULL == sg_logfile) {
        __writetips2console("open file error:%d %s, path:%s", errno, strerror(errno), sg_logfilename.c_str());
    }

    return NULL != sg_logfile;
}

static void __log2file(const void* _data, size_t _len) {
    if (NULL == _data || 0 == _len || sg_logdir.empty()) {
        return;
    }

	    ScopedLock lock_file(sg_mutex_log_file);
        if (__openlogfile(sg_logdir)) {
            __writefile(_data, _len);
            if (kAppednerAsync == sg_mode) {
                __closelogfile();
            }
        return;
}        
```
代码详见：[github](https://github.com/FelixZhang00/xlog-rotating-base-size)

---
## 相关链接
http://dev.qq.com/topic/581c2c46bef1702a2db3ae53
