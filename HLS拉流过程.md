# ZLM拉流(HLS)

### 一.总体流程图

[HLS拉流过程Zlm.drawio](./Images/HLS拉流过程Zlm.drawio.png)

### 二.M3U8请求过程

##### 1.MediaPlayer.cpp，继承自PlayerBase。在play函数中通过PlayerBase创建真正的播放delegate，设置响应的回调处理函数。设置成一个MediaSource。调用真正播放代理的play函数。

```c++
void MediaPlayer::play(const string &url) {
    _delegate = PlayerBase::createPlayer(_poller, url);
    assert(_delegate);
    setOnCreateSocket_l(_delegate, _on_create_socket);
    _delegate->setOnShutdown(_on_shutdown);
    _delegate->setOnPlayResult(_on_play_result);
    _delegate->setOnResume(_on_resume);
    _delegate->setMediaSource(_media_src);
    _delegate->mINI::operator=(*this);
    _delegate->play(url);
}
```

##### 2.PlayerBase.cpp,在createPlayer函数中通过传入的url字符串判断创建HLS播放器

```c++
PlayerBase::Ptr PlayerBase::createPlayer(const EventPoller::Ptr &poller, const string &url_in) {
    static auto releasePlayer = [](PlayerBase *ptr) {
        onceToken token(nullptr, [&]() {
            delete ptr;
        });
        ptr->teardown();
    };
    string url = url_in;

    if (url.find("#EXTM3U") == 0) {
        return PlayerBase::Ptr(new HlsPlayerImp(poller), releasePlayer);
    }

    string prefix = FindField(url.data(), NULL, "://");
    auto pos = url.find('?');
    if (pos != string::npos) {
        //去除？后面的字符串
        url = url.substr(0, pos);
    }
	/*
		其他Player判断代码
	*/

    throw std::invalid_argument("not supported play schema:" + url_in);
}
```

##### 3.HLSPlayer.cpp, 在fetchIndexFile中，调用TCPClient sendRequest方法，通过GET http请求获取m3u8文件内容，用于后续拉流。

```c++
void HlsPlayer::fetchIndexFile() {
    //_try_fetch_index_times = 0;
    if (waitResponse()) {
        return;
    }
    if (!(*this)[Client::kNetAdapter].empty()) {
        setNetAdapter((*this)[Client::kNetAdapter]);
    }
    setCompleteTimeout((*this)[Client::kTimeoutMS].as<int>());
    setMethod("GET");
    sendRequest(_play_url);
}
```

##### 4.TcpClient.cpp ,在startConnect中建立tcp链接，通过调用toolkit中network sockUtill相关方法创建套接字。

```c++
void TcpClient::startConnect(const std::string &url, uint16_t port, float timeout_sec, uint16_t local_port) {
    weak_ptr<TcpClient> weak_self = shared_from_this();

    _timer = std::make_shared<Timer>(2.0f, [weak_self]() {
        auto strong_self = weak_self.lock();
        if (!strong_self) {
            return false;
        }
        strong_self->onManager();
        return true;
    }, getPoller());

    setSock(createSocket());

    auto sock_ptr = getSock().get();
    sock_ptr->setOnErr([weak_self, sock_ptr](const SockException &ex) {
        auto strong_self = weak_self.lock();
        if (!strong_self) {
            return;
        }
        if (sock_ptr != strong_self->getSock().get()) {
            //已经重连socket，上次的socket的事件忽略掉
            return;
        }
        strong_self->_timer.reset();
        strong_self->onErr(ex);
    });

    sock_ptr->connect(url, port, [weak_self](const SockException &err) {
        auto strong_self = weak_self.lock();
        if (strong_self) {
            strong_self->onSockConnect(err);
        }
    }, timeout_sec, _net_adapter, local_port);
}
```

### 三.TS拉流过程

##### 1.HLSPlayer.cpp 在获取m3u8内容成功后，通过onResponseHeader和onResponseBody获取m3u8内容，获取完成后，通过onResponseCompleted进行下一步操作。

```c++
void HlsPlayer::onResponseHeader(const string &status, const HttpClient::HttpHeader &headers) {
    if (status != "200" && status != "206") {
        //失败
        throw invalid_argument("bad http status code:" + status);
    }
    auto content_type = strToLower(const_cast<HttpClient::HttpHeader &>(headers)["Content-Type"]);
    if (content_type.find("application/vnd.apple.mpegurl") != 0 && content_type.find("/x-mpegurl") == _StrPrinter::npos) {
        WarnL << "May not a hls video: " << content_type;
    }
    _m3u8.clear();
}

void HlsPlayer::onResponseBody(const char *buf, size_t size) {
    _m3u8.append(buf, size);
}

void HlsPlayer::onResponseCompleted(const SockException &ex) {
	if (0 == strcmp(ex.what(), "connection reset by peer"))
	{
		//通知TSDecoder将组帧缓冲清空，防止帧不完整导致花屏
		weak_ptr<HlsPlayer> weak_self = static_pointer_cast<HlsPlayer>(shared_from_this());
		auto strong_self = weak_self.lock();
		if (strong_self) {
			strong_self->onClearPacket();
		}
	}
	if (ex) {
        teardown_l(ex);
        return;
    }
    if (!HlsParser::parse(getUrl(), _m3u8)) {
        teardown_l(SockException(Err_other, "parse m3u8 failed:" + _play_url));
        return;
    }
    if (!_play_result) {
        _play_result = true;
        onPlayResult(SockException());
    }
    playDelay();
}
```

##### 2.HlsParser.cpp 获取m3u8内容完成后，调用HlsParser::parse方法解析m3u8内容，生成待播放的ts列表，激活onParsed回调。

```c++
bool HlsParser::parse(const string &http_url, const string &m3u8) {
 	//核心代码
    for (auto &line : lines) {
        /*
        	m3u8内容根据标准格式进行解析，组合成ts_list
        */
        continue;
    }

    if (_is_m3u8) {
        onParsed(_is_m3u8_inner, _sequence, ts_map);
    }
    return _is_m3u8;
}
```

##### 3.HlsPlayer.cpp,处理 HLS流媒体播放过程中解析M3U8文件的结果，对解析后的ts列表进行去重、缓存，调用fetchSegment进行拉流操作，并处理直播中断或M3U8更新异常。

```c++
void HlsPlayer::onParsed(bool is_m3u8_inner, int64_t sequence, const map<int, ts_segment> &ts_map) {
    //判断是否是ts播放列表
    //是
    for (auto &pr : ts_map) {
            auto &ts = pr.second;
            if (_ts_url_cache.emplace(ts.url).second) {
                // 该ts未重复
                _ts_list.emplace_back(ts);
                // 按时间排序
                _ts_url_sort.emplace_back(ts.url);
				//备份ts列表,用于seek请求时重新构造_ts_list
				_ts_list_back.emplace_back(ts);
            }
        }
        if (_ts_url_sort.size() > 2 * ts_map.size()) {
            // 去除防重列表中过多的数据
            _ts_url_cache.erase(_ts_url_sort.front());
            _ts_url_sort.pop_front();
        }

        InfoL << _ts_url_sort.size() << " ts segments in list";
       
        fetchSegment();
    //不是
    // 这是m3u8列表,我们播放最高清的子hls
        if (ts_map.empty()) {
            throw invalid_argument("empty sub hls list:" + getUrl());
        }
        _timer.reset();
        weak_ptr<HlsPlayer> weak_self = static_pointer_cast<HlsPlayer>(shared_from_this());
        auto url = ts_map.rbegin()->second.url;
        getPoller()->async([weak_self, url]() {
            auto strong_self = weak_self.lock();
            if (strong_self) {
                strong_self->play(url);
            }
        }, false);
}
```

##### 4.Hlsplayer.cpp,接上述步骤调用fetchSegment方法，该方法负责 HLS流媒体中TS分片的下载调度与管理，若当前无下载任务，从待下载队列 `_ts_list` 中取出首个TS分片的URL，发起HTTP请求下载。创建 HttpTsPlayer 实例处理HTTP请求，支持自定义Socket创建与数据包回调（OnPacket）。并计算合理的延迟Delay，下载下一个切片。

```c++
void HlsPlayer::fetchSegment() {
     if (_ts_list.empty()) {
     	//...
         	//播放列表为空，那么立即重新下载m3u8文件
         	_timer.reset();
            fetchIndexFile();
        //...
     }
    //创建HttpTsPlayer实例，处理http请求，创建socket，并定义数据包回调
    if (!_http_ts_player) {
        _http_ts_player = std::make_shared<HttpTSPlayer>(getPoller());
        _http_ts_player->setOnCreateSocket([weak_self](const EventPoller::Ptr &poller) {
            auto strong_self = weak_self.lock();
            if (strong_self) {
                return strong_self->createSocket();
            }
            return Socket::createSocket(poller, true);
        });
        auto benchmark_mode = (*this)[Client::kBenchmarkMode].as<int>();
        if (!benchmark_mode) {
            _http_ts_player->setOnPacket([weak_self](const char *data, size_t len) {
                auto strong_self = weak_self.lock();
                if (!strong_self) {
                    return;
                }
                //收到ts包
                strong_self->onPacket(data, len);
            });
        }

        if (!(*this)[Client::kNetAdapter].empty()) {
            _http_ts_player->setNetAdapter((*this)[Client::kNetAdapter]);
        }
    }
    _http_ts_player->setOnComplete([weak_self, ticker, duration, url](const SockException &err) {
        /*
    		Delay计算，省略
    	*/
            //延时下载下一个切片
        strong_self->_timer_ts.reset(new Timer(delay / strong_self->_speed, [weak_self]() {
            auto strong_self = weak_self.lock();
            if (strong_self) {
			   InfoL << "Download ts segment fetchSegment";

                strong_self->fetchSegment();

            }
            return false;
        }, strong_self->getPoller()));
    });
    _http_ts_player->setMethod("GET");
    //ts切片必须在其时长的2-5倍内下载完毕
    _http_ts_player->setCompleteTimeout(_timeout_multiple * duration * 1000);
    _http_ts_player->sendRequest(url);
}
```

##### 5.在onPacket中进行创建解码器（TSDecoder），并解码的操作，具体流程省略。

```c++
void HlsPlayerImp::onPacket(const char *data, size_t len) {
    if (!_decoder && _demuxer) {
        _decoder = DecoderImp::createDecoder(DecoderImp::decoder_ts, _demuxer.get());
    }

    if (_decoder && _demuxer) {
        _decoder->input((uint8_t *) data, len);
    }
}
```

##### 6.解码成功后通过onFrame回调给Demuxer层。

```c++
void DecoderImp::onFrame(const Frame::Ptr &frame) {
	if (_sink)
	{
		_sink->inputFrame(frame);
	}
}
```

##### 7.再通过HlsDemuxer::inputframe调用到MediaSink::inputframe，MediaSink 类是一个 **核心的多媒体数据接收与分发器**，负责将音视频帧数据分发到不同的处理模块或输出目标。MediaSink::inputFrame会找到对应的Track并调用其inputFrame，Track继承自FrameDispatcher，所以最终会调用FrameDispatcher::inputFrame。

```c++
bool MediaSink::inputFrame(const Frame::Ptr &frame) {
    //查找对应轨道并分发
    auto it = _track_map.find(frame->getTrackType());
    if (it == _track_map.end()) {
        return false;
    }
    //got frame
    it->second.second = true;
    auto ret = it->second.first->inputFrame(frame);
    if (_mute_audio_maker && frame->getTrackType() == TrackVideo) {
        //视频驱动产生静音音频
        _mute_audio_maker->inputFrame(frame);
    }
    checkTrackIfReady();
    return ret;
}
```

##### 8.回到最上层，调用mk_track_add_delegate 注册回调到FrameDispatcher，供后续使用。它创建一个FrameWriterInterfaceHelper对象，并通过Track::addDelegate添加到Track的委托列表,当FrameDispatcher::inputFrame执行时，会遍历这些委托并调用它们的inputFrame方法.

```c++
//自定义处理回调
commonIo->_tracks[1] = mk_track_ref(tracks[i]);
mk_track_add_delegate(tracks[i], [](void* user_data, mk_frame frame) {
    CommonIo* commonIo = reinterpret_cast<CommonIo*>(user_data);
    commonIo->inputFrame(frame);
}, commonIo);
```

##### 9.注册之后，在FrameDispatcher::inputFrame中调用这些注册的回调。到这一步，媒体数据就从ZLMediaKit内部传递到了您的应用代码中。

```c++
bool inputFrame(const Frame::Ptr &frame) override {
    std::lock_guard<std::mutex> lck(_mtx);
    bool ret = false;
    for (auto &pr : _delegates) {
        if (pr.second->inputFrame(frame)) {
            ret = true;
        }
    }
    return ret;
}
```

