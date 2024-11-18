+++
title = '小小的GoTo语句'
date = 2024-11-18T23:24:04+08:00
categories = ['Study']
keywords = ['goto']
draft = false
summary = '半夜突然翻阅到一篇很经典的关于[GoTo文章](https://www.emon100.com/goto-translation/)，对于其中一个很典型的函数示例感受颇深，因此对此进行记录！'

+++

# 1. goto语句示例

```c
int parse()
{
	Token tok;
reading:
  tok = gettoken();
  if (tok == END)
    return ACCEPT;
shifting:
  if (shift(tok))
    goto reading;
reducing:
  if (reduce(tok))
    goto shifting;
  return ERROR;
}
```

# 2. 不使用goto语句情况

> 下面是来自ChatGPT的回答！脑袋笨，想不到ChatGPT这一层，看完恍然大悟！也可能是忘记了`shift`和`reduce`的含义，也可能是深夜困了！

## 2.1 示例1

```c
int parse()
{
    Token tok;
    while (true)
    {
    		// Reading state
        tok = gettoken();
        if (tok == END)
            return ACCEPT;
        while (true)
        {
            // Shifting state
            if (shift(tok))
                break; // 回到外层循环继续读取 token
            // Reducing state
            if (reduce(tok))
                continue; // 继续尝试移位
            return ERROR; // 无法归约时返回错误
        }
    }
}
```

## 2.2 示例2

```C
int parse()
{
    Token tok;
    enum State { READING, SHIFTING, REDUCING } state = READING;

    while (true)
    {
        switch (state)
        {
        case READING:
            tok = gettoken();
            if (tok == END)
                return ACCEPT;
            state = SHIFTING;
            break;

        case SHIFTING:
            if (shift(tok))
            {
                state = READING;
            }
            else
            {
                state = REDUCING;
            }
            break;

        case REDUCING:
            if (reduce(tok))
            {
                state = SHIFTING;
            }
            else
            {
                return ERROR;
            }
            break;
        }
    }
}
```

## 2.3 示例3

```c
int parse()
{
    Token tok;
    auto reading = [&]() -> bool {
        tok = gettoken();
        return tok != END;
    };

    auto shifting = [&]() -> bool {
        return shift(tok);
    };

    auto reducing = [&]() -> bool {
        return reduce(tok);
    };

    while (true)
    {
        if (!reading())
            return ACCEPT;

        while (true)
        {
            if (shifting())
                break;

            if (!reducing())
                return ERROR;
        }
    }
}
```

