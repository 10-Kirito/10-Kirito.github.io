+++
title = 'CodeReview(ClusterChatServer)'
date = 2024-11-20T20:20:04+08:00
tag = ['CodeReview']
categories = ['CodeReview']
keywords = ['CodeReview', 'ChatServer']
summary = '[CodeReview-ClusterChatServer](https://github.com/10-Kirito/ClusterChatServer) - 本次CodeReview的主题是ClusterChatServer，主要是对ClusterChatServer的代码进行Review，查看代码的质量，是否符合规范，是否有潜在的问题。该项目是本人第一个完整的C++项目，当初编写的时候经验不是很多，所以一定存在一定的问题。但是一直没有对这一坨屎山进行CodeReview，希望本次CodeReview可以给你一些帮助！'
draft = false
+++

> [ClusterChatServer](https://github.com/10-Kirito/ClusterChatServer)为本人大四简历上面的一个项目，实现的是一个简单的集群聊天服务器，其中支持一对一以及多对多的聊天功能！如果你对该项目感兴趣的话，可以在GitHub上进行查看。

# 1. 存在的问题总结

## 1.1 可见域问题

> 该问题可能是工作之前很少注意到吧！记得刚开始工作的时候也遇到过一些这样的问题，当时是被同事纠正过来！

***第一处：***

```C++
class ChatService : public SingletonTemplate<ChatService> {
public
	void Login(const TcpConnectionPtr &, json &, Timestamp);
  void Register(const TcpConnectionPtr &, json &, Timestamp);
  void OneChat(const TcpConnectionPtr &, json &, Timestamp);
  void GroupChat(const TcpConnectionPtr &, json &, Timestamp);
  void AddFriend(const TcpConnectionPtr &, json &, Timestamp);
  void QueryFriends(const TcpConnectionPtr &, json &, Timestamp);
  void CreateGroup(const TcpConnectionPtr &, json &, Timestamp);
  void DeleteGroup(const TcpConnectionPtr &, json &, Timestamp);
  void JoinGroup(const TcpConnectionPtr &, json &, Timestamp);
  void QuitGroup(const TcpConnectionPtr &, json &, Timestamp);
  void UpdateUser(const TcpConnectionPtr &, json &, Timestamp);
  // 对外暴露的接口只有这两个
  MessageHandler getHandler(MessageType type);
  void clientCloseException(const TcpConnectionPtr &connection);
private
  std::unordered_map<MessageType, MessageHandler> _msgHandlerMap;
}
```

这些接口外部并不是直接使用这些借口的，而是通过`getHandler`来访问的，所以这些接口的可见域应该为`private`。

## 1.2 接口实现过于过程化

```C++
void ChatService::Login(const TcpConnectionPtr &connection, json &data,
                        Timestamp time) {
  LOG_INFO << "Checking Login...";
  int id = data["id"];
  std::string pwd = data["password"];

  User user = _userModel.query(id);

  json response;
  response["msgtype"] = MessageType::LOGIN_MSG_ACK;
  // LOG_INFO << "Checking user's password...";
  if (user.getId() != -1 && user.getPwd() == pwd) {
    if (user.getState() == "online") {
      // the user is already online
      LOG_INFO << "Login failed, the user is already online!";
      response["status"] = 401;
      response["errorMessage"] = "The user already login!";
    } else {
      // login successfully and store the user's connection
      // !!!need to consider the concurrency problem
      {
        std::lock_guard<std::mutex> lock(_mutex);
        _userConnMap.insert({id, connection});
        _userMap.insert({connection, id});
      }
      // begin to send the message to the user
      LOG_INFO << "Login successful!";
      user.setState("online");
      _userModel.update(user);
      response["status"] = 200;
      response["id"] = user.getId();
      response["name"] = user.getName();

      // 1. get the offline message from the database
      std::vector<Message> old_message = _messageModel.queryMsg(user.getId());
      if (!old_message.empty()) {
        response["offlinemsg"] = old_message;
      }
      // delete all the offline message, because the user has already read them
      _messageModel.deleteMsg(old_message);
      // 2. get the user's all friends
      std::vector<User> friends = _friendModel.query(user.getId());
      if (!friends.empty()) {
        response["friends"] = friends;
      }
      // 3. get the user's all groups
      std::vector<Group> groups = _groupModel.queryGroups(user.getId());
      if (!groups.empty()) {
        response["groups"] = groups;
        // get the user's all groups' messages
        for (const auto &group : groups) {
          std::vector<Message> old_group_message =
              _messageModel.queryGroupMsg(group.getId(), user.getId());
          if (!old_group_message.empty()) {
            response["groupmsg"].push_back(old_group_message);
            _messageModel.deleteGroupMsg(old_group_message);
          }
        }
      }
    }
  } else {
    LOG_INFO << "Login failed, the password is Wrong!";
    response["status"] = 400;
    response["errorMessage"] =
        "The password is wrong or the user is not exist!";
  }
  // std::cout << std::setw(4) << response;
  connection->send(response.dump());
}
```

你能想象这是一个登陆接口的实现吗？这里完全可以拆分为几个函数来实现，否则就会出现在公司当初见到的那个近千行的函数实现！

> 这里当时我在公司还吐槽当时实现的人怎么会写出这样的代码，现在一看当初的我也是这么干的，就这么将整个登陆路程讲述了一遍！

对上面的代码简化后:

```C++
void ChatService::Login(const TcpConnectionPtr &connection, json &data, Timestamp time) {
    LOG_INFO << "Checking Login...";

    // Parse user credentials
    int id = data["id"];
    std::string pwd = data["password"];
    json response;
    response["msgtype"] = MessageType::LOGIN_MSG_ACK;

    // Query user from database
    User user = _userModel.query(id);
    if (!ValidateUser(user, pwd, response)) {
        connection->send(response.dump());
        return;
    }

    // Check if user is already online
    if (user.getState() == "online") {
        HandleAlreadyOnline(response);
        connection->send(response.dump());
        return;
    }

    // Handle successful login
    HandleSuccessfulLogin(user, connection, response);

    connection->send(response.dump());
}

// 验证用户
bool ChatService::ValidateUser(const User &user, const std::string &pwd, json &response) {
    if (user.getId() == -1 || user.getPwd() != pwd) {
        LOG_INFO << "Login failed, the password is Wrong!";
        response["status"] = 400;
        response["errorMessage"] = "The password is wrong or the user does not exist!";
        return false;
    }
    return true;
}

// 处理用户已经在线的情况
void ChatService::HandleAlreadyOnline(json &response) {
    LOG_INFO << "Login failed, the user is already online!";
    response["status"] = 401;
    response["errorMessage"] = "The user is already logged in!";
}

// 处理登录成功的逻辑
void ChatService::HandleSuccessfulLogin(User &user, const TcpConnectionPtr &connection, json &response) {
    LOG_INFO << "Login successful!";

    // Update user state and store connection
    {
        std::lock_guard<std::mutex> lock(_mutex);
        _userConnMap[user.getId()] = connection;
        _userMap[connection] = user.getId();
    }

    user.setState("online");
    _userModel.update(user);

    response["status"] = 200;
    response["id"] = user.getId();
    response["name"] = user.getName();

    // Populate user data
    PopulateOfflineMessages(user, response);
    PopulateFriendList(user, response);
    PopulateGroupData(user, response);
}

// 获取用户的离线消息
void ChatService::PopulateOfflineMessages(const User &user, json &response) {
    std::vector<Message> oldMessages = _messageModel.queryMsg(user.getId());
    if (!oldMessages.empty()) {
        response["offlinemsg"] = oldMessages;
        _messageModel.deleteMsg(oldMessages);
    }
}

// 获取用户的好友列表
void ChatService::PopulateFriendList(const User &user, json &response) {
    std::vector<User> friends = _friendModel.query(user.getId());
    if (!friends.empty()) {
        response["friends"] = friends;
    }
}

// 获取用户的群组数据及群组消息
void ChatService::PopulateGroupData(const User &user, json &response) {
    std::vector<Group> groups = _groupModel.queryGroups(user.getId());
    if (!groups.empty()) {
        response["groups"] = groups;
        for (const auto &group : groups) {
            std::vector<Message> groupMessages = _messageModel.queryGroupMsg(group.getId(), user.getId());
            if (!groupMessages.empty()) {
                response["groupmsg"].push_back(groupMessages);
                _messageModel.deleteGroupMsg(groupMessages);
            }
        }
    }
}

```

虽然看着比原来的代码更长了一些，但是逻辑清晰了不止一星半点！

# 2. 有趣的写法

> 这里说来惭愧，因为工作之后经常使用的编程语言是Delphi，自己对于C++的学习便一直处于停滞不前的状态，所以本次CodeReview的时候很多写法我都看不懂！但是看懂之后，C++不愧是C++，感觉就是和其他的语言不一样！

## 2.1 单例模式和友元类

```C++
template <typename T> class SingletonTemplate {
public:
  static T &getInstance() {
    static T instance;
    return instance;
  }
  virtual ~SingletonTemplate() = default;
  // delete copy constructor and assignment operator
  SingletonTemplate(const SingletonTemplate &) = delete;
  SingletonTemplate &operator=(const SingletonTemplate &) = delete;
protected:
  SingletonTemplate() = default;
};

class ChatService : public SingletonTemplate<ChatService> {
  // friend class of SingletonTemplate, and SingletonTemplate<ChatService> can
  // access the private members of ChatService
  friend class SingletonTemplate<ChatService>; 
}
```

这里结合泛型模版以及友元机制来使得单例类的实现十分的简洁！

其中单例模版类，其中将构造函数声明为`protected`，不能被外界所使用，并且将拷贝构造函数以及赋值函数删除，以及通过提供一个静态的获取单例的函数来实现单例模式。其中由于`static`的一些特性，保证了一定只构建一个单例。而且与此同时利用C++11以及以后标准当中对于静态变量初始化的线程安全保证。

> C++11 标准开始，对函数内部的静态局部变量（如 `static T instance`）的初始化提供了以下保证：
>
> - **仅初始化一次**：无论有多少线程尝试访问 `getInstance`，`static T instance` 的初始化只会发生一次。
> - **线程安全**：多个线程同时进入 `getInstance` 时，初始化会被 C++ 编译器正确同步，确保没有竞争条件。

回顾这段代码的时候，唯一令我困惑的是为什么这里需要将`SingletonTemplate<ChatService`类声明为`ChatService`的友元类？原来是这样可以保证类`SingletonTemplate<ChatService>`可以访问被隐藏的`ChatService`的构造函数。

## 2.2 C++当中打印枚举类型值

```C++
enum class MessageType {
  // ...
}

std::ostream &operator<<(std::ostream &out, const MessageType type) {
  static const auto strings = [] {
    std::map<MessageType, std::string_view> result;
#define INSERT_ELEMENT(p) result.emplace(p, #p);
    INSERT_ELEMENT(MessageType::LOGIN_MSG);
    INSERT_ELEMENT(MessageType::REGIST_MSG);
#undef INSERT_ELEMENT
    return result;
  };
  return out << strings()[type];
}
```

这里主要的困惑之处在于`result.emplace(p, #p)`这里的作用。

> **宏的作用**：简化代码，将枚举值和其字符串表示添加到 `result` 中。
>
> - `emplace` 的含义是直接在 `std::map` 中插入一个键值对；
> - `#p` 是宏预处理中的字符串化操作，用于将 `p` 转换为字符串。例如：`#MessageType::LOGIN_MSG` 会变为 `"MessageType::LOGIN_MSG"`；

​	
