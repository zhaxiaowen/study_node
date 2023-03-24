# python生成验证码

> ```
> pip install captcha
> ```

```
from captcha.image import ImageCaptcha
from random import randint

def create_image(file_name):
    list = ['0', '1', '2', '3', '4', '5', '6', '7', '8', '9']

    data = ''
    for i in range(4):
        data += list[randint(0, 9)]

    image = ImageCaptcha().generate_image(data)
    # image.show()
    image.save("D:\\Project\\picture\\{0}.png".format(file_name))

for i in range(5):
    create_image(i)
```

