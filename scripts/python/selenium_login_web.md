# selenium自动登录网站

#### 环境准备

```
chrome 驱动下载地址 :  https://www.jb51.net/article/140239.htm

pip3 install selenium==3.3.0
pip3 install pytesseract

tesseract 参考教程 :
https://blog.csdn.net/weixin_51571728/article/details/120384909
tesseract 下载地址 :
https://digi.bib.uni-mannheim.de/tesseract/

语言库下载地址 :
https://github.com/tesseract-ocr/tessdata
```



#### 不带验证码的

```
from selenium import webdriver
import time
import os

def auto_login(url_, username_, pwd_, uid_xpath_, pwd_xpath_, button_xpath_):
    chromedriver = "D:\Download\chromedriver_win32\chromedriver.exe"  # 这里写本地的 chromedriver 的所在路径
    os.environ["webdriver.Chrome.driver"] = chromedriver  # 调用 chrome 浏览器
    driver = webdriver.Chrome(chromedriver)  ##设置 Chrome 驱动
    driver.maximize_window()  ##最大化窗口
    driver.implicitly_wait(6)  ##隐式等待
    driver.get(url_)  ##获取网页
    time.sleep(1)
    driver.find_element_by_xpath(uid_xpath_).clear()  ##清空已存在内容
    driver.find_element_by_xpath(uid_xpath_).send_keys(username_)  ##通过第 2 步的用户名 xpath 找到用户名文本框并传入用户名
    driver.find_element_by_xpath(pwd_xpath_).clear()  ##清空已存在内容
    driver.find_element_by_xpath(pwd_xpath_).send_keys(pwd_)  ##过第 2 步的密码 xpath 找到密码文本框并传入密码
    time.sleep(1)
    driver.find_element_by_xpath(button_xpath_).click()  ##过第 2 步的登录按钮 xpath 找到登录按钮并单击登录
    time.sleep(5)

auto_login(url_="https://apollo-offline-wyc.wsecar.cn/signin", username_="apollo_wyc_offline", pwd_="9tADjBlEdiz72_OWjrRSCaw9BCMA1S", uid_xpath_='.//input[@name="username"]',
           pwd_xpath_='.//input[@name="password"]', button_xpath_='.//input[@name="login-submit"]')

```

#### 带验证码,但是识别率很低

```
from selenium import webdriver
from PIL import Image
import pytesseract
import os,time
chromedriver = "D:\Download\chromedriver_win32\chromedriver.exe" #这里写本地的 chromedriver 的所在路径
os.environ["webdriver.Chrome.driver"] = chromedriver #调用 chrome 浏览器
driver = webdriver.Chrome(chromedriver)
driver.get("https://sysmanager-fat06.wsecar.com/") #该处为具体网址
driver.refresh() #刷新页面
driver.maximize_window() #浏览器最大化
driver.implicitly_wait(6)  ##隐式等待
#获取全屏图片，并截取验证码图片的位置
driver.get_screenshot_as_file('a.png')
location = driver.find_element_by_xpath('/html/body/app-root/nz-spin/div/div[2]/div/login/nz-spin/div/div[2]/div/div/div/div[2]/img').location
size = driver.find_element_by_xpath('/html/body/app-root/nz-spin/div/div[2]/div/login/nz-spin/div/div[2]/div/div/div/div[2]/img').size
print(location)
print(size)

left = location['x']
top = location['y']
right = location['x'] + size['width']
bottom = location['y'] + size['height']
a = Image.open("a.png")
im = a.crop((left,top,right,bottom))
im.save('a.png')
time.sleep(1)
#打开保存的验证码图片
image = Image.open("a.png")
#图片转换成字符
vcode = pytesseract.image_to_string(image)
print(vcode)
#填充用户名 密码 验证码
vcode=input(" 请输入验证码 ")
driver.find_element_by_xpath("/html/body/app-root/nz-spin/div/div[2]/div/login/nz-spin/div/div[2]/div/div/div/div[2]/input[1]").send_keys("18924175973")
driver.find_element_by_xpath("/html/body/app-root/nz-spin/div/div[2]/div/login/nz-spin/div/div[2]/div/div/div/div[2]/input[2]").send_keys("18924175973")
driver.find_element_by_xpath("/html/body/app-root/nz-spin/div/div[2]/div/login/nz-spin/div/div[2]/div/div/div/div[2]/input[3]").send_keys(vcode)

#点击登录
driver.find_element_by_class_name("login-btn").click()
```



#### 测试文字库

```
import pytesseract
from PIL import Image
def demo():
    # 打开要识别的图片
    image = Image.open('E:\\1.png')
    image=image.convert('1')
    # threshold = 150
    # table = []
    # for i in range(256):
    #     if i < threshold:
    #         table.append(0)
    #     else:
    #         table.append(1)
    # # image = image.convert('RGB')
    # image=image.point(table,"1")
    # # print(image)
    # image.show()
    # 使用 pytesseract 调用 image_to_string 方法进行识别，传入要识别的图片，lang='chi_sim' 是设置为中文识别，
    text = pytesseract.image_to_string(image,lang='chi_sim')
    # 输入所识别的文字
    print(text)
if __name__ == '__main__':
    demo()

```

#### tesseract-3.95.01可以用命令试试,但是也不准确

```
tesseract.exe D:\Project\3.png D:\Project\tmp_dir\1 -psm 50
```

#### [提高tesseract识别准度](https://www.mspring.org/tags/tesseract/)

```
1.Tesseract 更新到 4.0 以上版本
2. 图像前处理， 去噪，二值化
3. 这个是最有效的，文字分行分块处理， 调用 psm=PSM.SINGLE_BLOCK
4. 自己训练，4.0 可以 fine tuning
```

