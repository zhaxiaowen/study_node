# cv2图片处理

#### 图片放大缩小

> 指定任意尺寸

```
import cv2
import matplotlib.pylab as plt
img = cv2.imread("D:\\Project\\picture\\0.png",flags=cv2.IMREAD_GRAYSCALE)
h,w=img.shape[:2]
img = cv2.resize(img,dsize=(w*10,h*10))
cv2.imshow("Image",img)
cv2.waitKey (0)

# plt.subplot(1, 2, 1)
# plt.imshow(img)
# plt.subplot(1, 2, 2)
# plt.imshow(img2)
# plt.show()
```

> 根据坐标轴系数进行缩放

```
import cv2
import matplotlib.pylab as plt

image = cv2.imread('D:\\Project\\picture\\0.png')
image_resize = cv2.resize(image, None, fx=0.5, fy=0.5)
print(image.shape)

plt.subplot(1, 2, 1)
plt.imshow(image)
plt.subplot(1, 2, 2)
plt.imshow(image_resize)
plt.show()
```

