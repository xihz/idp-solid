# IDP

## SOLID

### Single Responsibility

> "Class should only have 1 reason to change".

While designing class, it should be as concise as possible. This also relates to 'high-cohesion' quality
In following example we have class CatUploader carrying out extra actions like uploading images.

```C++
class CatPic;

class CatUploader
{
public:

    const CatPic& GetCatPic(const Camera& cam)
    {
        cat_pic = cam.GetImage();
        m_cat_pics.emplace_back(cat_pic);
        return cat_pic;
    }

    CatPic CropCatPicture(const CatPic& cat_pic, const Point& top_left, const Point& bottom_right)
    {
        return cat_pic.Crop(top_left, bottom_right)
    }

    void UploadPic(const MyCloud& cloud)
    {
        for (const auto& pic : m_cat_pics)
        {
            cloud.upload(pic.GetImageBuffer())
        }
    }


private:
    std::vector<CatPic> m_cat_pics;
};

```

The better solution is to separate method and group them up in separate entities. Now we have Storing/Processing/Creating operations assorted by usage

``` C++
class CatPic;
using CatPicVector = std::vector<CatPic>;

class CatPicProcessor()
{
    CatPic CropCatPicture(const CatPic& cat_pic);
}

class CatPicStorage
{
    void AddPic(const CatPic& pic)
    {
        m_pics.emplace_back(pic);
    }

    CatPicVector& GetPic()
    {
        return m_pics;
    }

private:
    CatPicVector m_pics;
};

class CatCamera()
{
    CatPic GetCatPic()
    {
        return m_camera.GetImage()
    }
private:
    Camera m_camera;
};

class CatUploader()
{
    ICatUploader(const MyCloud& cloud, const CatPicStorage& storage;): m_cloud(cloud), m_storage(storage)
    {

    }

    void Upload()
    {
        for (const auto& pic : storage.GetPics())
        {
            m_cloud.upload(pic.GetData());
        }
    }
private:
    MyCloud& m_cloud;
    CatPicStorage& storage;
};
```

### Open-Close Principe

> "Software entities ... should be open for extension, but closed for modification."

Its better to design software extensible, to allow possible or planned seamless extension in future.
One of the interpretations of this principe can encourage to provide wrappers/adapters for 3rd party modules rather than modifying them directly

In following example we have some notification service and we're extending it with different channels.
The problem is: we need to add new implementation to relatively high level client code with every new notification method,

```C++
class MailLib;
class SlackLib;
class PushLib;

class ServiceNotifier
{
    //1st iteration,
    NotifyMail(MailLib& lib, const std::string_view message);
    // added channel
    NotifySlack(SlackLib& lib, const std::string_view message);
    // and even more
    NotifyPush(PushLib& lib, const std::string_view message);
};
```

We can improve this by providing interface-based implementation. Now we need to implement new channel and register it.

```C++
class INotify
{
    virtual void Notify(const std::string_view message) = 0;
    virtual ~INotify() = default;
}

class MailNotifier: public INotify
{
public:
    void Notify(const std::string_view message)
    {
        m_lib.send_msg(message)
    }
private:
    MailLib m_lib;
};

class SlackNotifier: public INotify
{
public:
    void Notify(const std::string_view message)
    {
        m_lib.post_message(message)
    }
private:
    SlackLib m_lib;
};


using NotifyPtr = std::unique_ptr<Inotify>;
class ServiceNotifier()
{
public:
    void AddNotifier(NotifyPtr notifier)
    void NotifyAll(const std::string_view message)
    {
        for(const auto& notifier: m_notifiers)
        {
            notifier->Notify(message);
        }
    }

private:
    std::vector<NotifyPtr> m_notifiers;
}

//client code:
service = ServiceNotifier();
service.AddNotifier(slackNotifier);
service.NotifyAll();
```

### Liskov

> "If [S] is a subtype of [T], then objects of type [T] in a program may be replaced with objects of type [S] without altering any of the desirable properties of that program (e.g. correctness)"

While designing class hierarchy, we need to make sure all deriving classes will act like its parent.
In the following example we have poorly designed hierarchy, leading to inconsistent behavior of Rooster class

``` C++
class IBird
{
    virtual void RaiseWings() = 0;
    virtual void void Fly() = 0;
    virtual ~IBird() = 0
};

class Duck : public IBird
{
    void RaiseWings() override;
    void void Fly() override;
}

class Rooster : public IBird
{
    void RaiseWings() override;
    void void Fly() override
    {
        throw std::runtime_error("Ooops, I cant fly:(")
    }
}

```

We can improve this be specifying "parents", now we can pass IBird derived class safely for cases with no fly requirements.

``` C++
class IBird
{
    virtual void RaiseWings() = 0;
    virtual ~IBird() = 0
};

class IFlyingBird : public IBird
{
    virtual void Fly() = 0;
    virtual ~IFlyingBird() = 0
}

class Duck : public IFlyingBird
{
    void RaiseWings() override;
    void void Fly() override;
}

class Rooster : public IBird
{
    void RaiseWings() override;
}
```

### Interface segregation

> Many client-specific interfaces are better than one general-purpose interface.

We shouldn't force user to implement unnecessary interface parts.

In this example we have one broad interface, but its to wide for old player class

```C++
class IPlayer:
{
public:
    virtual void Play() = 0;
    virtual void Stop() = 0;
    virtual void FastForward() = 0;

    virtual void RegisterCloudAccount(const std::string_view account_name) = 0;
    virtual void SyncToCloud() = 0;
};


class IPadProMaxPlus : public IPlayer
{
    // play, stop, ff impl
};


class OldSonyWalkman: public IPlayer
{
public:
    // play, stop, ff impl

    void RegisterCloudAccount(const std::string_view account_name) override
    {
        throw std::logic_error("there is no cloud in 1990s");
    }
    void SyncToCloud() override
    {
        throw std::logic_error("there is no cloud in 1990s");
    }
};

```

The better solution is to provide two concise interfaces for each action group:

```C++
class ITrackControl
{
public:
    virtual void Play() = 0;
    virtual void Stop() = 0;
    virtual void FastForward() = 0;
};

class ICloudServices
{
public:
    virtual void RegisterCloudAccount(const std::string_view account_name) = 0;
    virtual void SyncToCloud() = 0;
};

class IPadProMaxPlus : public ITrackControl, public ICloudServices
{
};

class OldSonyWalkman : public ITrackControl
{
};

```

### Dependency Inversion

> Depend upon abstractions, [not] concretions.

Adding layer of abstraction can hide implementation details, allowing to change it without client-side code modifications. This principe is also related to 'loose coupling' system quality.

In this example we have client aware of implementation details(e.g how to check return value for errors)
It will require changes if we decide to swap underlying lib for another one.

```C++
class CloudUploader
{
public:
    CURLcode Upload(const std::string_view local_path, const std::string_view remote_path)
    {
        // get data from local_path;
        curl_easy_setopt(curl, CURLOPT_URL, remote_path);
        CURLcode result = curl_easy_perform(curl);
        curl_easy_cleanup(curl);
        return result;
    }

};

uploader = CloudUploader();
result = uploader.Upload('local', 'remote');
if(result != CURL_OK) // need to change condition in case of winhttp implementation
{
    /* handle the error case */
}

```

In simplest case we are not introducing new abstraction layer, but hiding implementation details.
Now we can easily change implementation without modifying client code.

``` C++
class CloudUploader
{
    // implementation1
    bool Upload(const std::string_view local_path, const std::string_view remote_path) override
    {
        CURL *curl = curl_easy_init();
        // get data from local_path;
        curl_easy_setopt(curl, CURLOPT_URL, remote_path);
        CURLcode res = curl_easy_perform(curl);
        curl_easy_cleanup(curl);

        return res == CURLE_OK;
    }
    // implementation2
    bool Upload(const std::string_view local_path, const std::string_view remote_path) override
    {
        HINTERNET hSession = WinHttpOpen(..);
        HINTERNET hConnect = WinHttpConnect( hSession, ...);
        HINTERNET hRequest = WinHttpOpenRequest(hConnect, ...);
        WinHttpSendRequest(hRequest, ...);
        WinHttpWriteData(hRequest, ...);
        result = WinHttpReceiveResponse( hRequest, NULL);
        if (hRequest) WinHttpCloseHandle(hRequest);
        if (hConnect) WinHttpCloseHandle(hConnect);
        if (hSession) WinHttpCloseHandle(hSession);
        return result == TRUE;
    }
};

cloud_uploader = CloudUploader();
if (!uploader.Upload('local', 'remote'))
{
    //error handling
}
```
