# Random BMP Viewer

This project is a simple BMP image viewer that I built using PyQt5.  
It parses uncompressed BMP files (1bpp, 4bpp, 8bpp, and 24bpp) manually, converts the raw byte data into a pixel grid, and renders the image onscreen.  
The viewer supports interactive controls like brightness, scaling, and RGB channel toggles, all done without using external image libraries. Its important to note that while I did use the QImage object, I only used it as a canvas to display pixels on, as the only other method Qt permits for this, would be to create an OpenGl window/context and write fragment shaders. 100% of the image operations are done manually.

---

## Overview

When a BMP file is dropped into the window:
1. The program parses the file header and color table.
2. It converts the pixel data into a 2D grid of RGB tuples.
3. The grid is displayed on screen using `QImage`.
4. Adjustments like brightness, scaling, and color channel masking are applied through a small image-processing pipeline.


---

##  Core Components

### `FileDrop`
```python
class FileDrop(QWidget):  
    dropped = pyqtSignal(str)  
    def __init__(self):  
        super().__init__()  
        self.setAcceptDrops(True)
        ...(rest of the class)
```
Qt widget that handles drag-and-drop interaction for BMP files.

- Accepts a single `.bmp` file.
- Highlights visually when a valid file is dragged over.
- Emits the file path to the main window when dropped.

---

### `BMPFile`
Object/class thats responsible for parsing and decoding the BMP file structure.

- Reads the **file header**, **DIB header**, and **color table** directly from binary.
- Supports the required bits per pixel:
  - 1bpp — black/white (or two other colors if listed in color table)
  - 4bpp — 16-color palette
  - 8bpp — 256-color palette
  - 24bpp — true color
- This class parses converts the bitmap data into a 2D grid of `(R, G, B)` tuples which is passed onto the ImageViewer object for rendering

Each `_parse_[Insert_bpp_Here]bpp()` function handles its own format, reconstructing the correct pixel values based on bit depth and palette.
  
```python
class BMPFile:  
    def __init__(self, url):  
        self.url = url  
        self.bytes = None  
  self.fileSize = 0
  ... (rest of the class)
```


#### Parsing Functions
Each parsing function handles a specific bit depth of BMP format.  
They all follow the same basic pattern:
1. Compute the row stride (BMP rows are padded to 4-byte boundaries).
2. Decide whether the image is top-down or bottom-up based on the header.
3. Loop through every pixel, decode it according to bit depth, and store it as `(R, G, B)` tuples.

**`_parse_1bpp()`**  
- Each pixel is a single bit  0 or 1, meaning each byte holds 8 pixels.  
- The bit value determines which of the two palette entries to use (usually black or white).  
- Used mostly for monochrome or dithered images.

```python
    def _parse_1bpp(self):  
        if len(self.colorTable) != 2:  
            print(f"Expected 2 color table entries for 1-bit BMP, got {len(self.colorTable)}")  
            exit(1)  
      
        stride = self._row_stride()  
        H = self._abs_height()  
        top_down = self._is_top_down()  
      
        color0 = self.colorTable[0]  
        color1 = self.colorTable[1]  
      
        grid = []  
        for y in range(H):  
            src_row = y if top_down else (H - 1 - y)  
            row_offset = self.dataOffset + src_row * stride  
            row_pixels = []  
            for x in range(self.width):  
                byte_index = x // 8  
      bit_index = 7 - (x % 8)  
                byte_value = self.bytes[row_offset + byte_index]  
                bit = (byte_value >> bit_index) & 1  
      c = color1 if bit else color0  
                row_pixels.append((c[0], c[1], c[2]))  
            grid.append(row_pixels)  
      
        self.pixelmap = grid  
        return grid
```

**`_parse_4bpp()`**  
- Every byte stores two pixels, one in the high nibble (4 bits) and one in the low nibble.  
- Each 4-bit value is an index into the color table (up to 16 colors).  
- Requires bit masking and shifting to extract the right nibble per pixel.
```python
    def _parse_4bpp(self):  
        if len(self.colorTable) == 0:  
            print("4bpp BMP requires a color table (palette)")  
            exit(1)  
      
        stride = self._row_stride()  # ((bpp*width + 31)//32)*4  
      H = self._abs_height()  
        top_down = self._is_top_down()  
      
        grid = []  
        for y in range(H):  
            src_row = y if top_down else (H - 1 - y)  
            row_offset = self.dataOffset + src_row * stride  
            row_pixels = []  
      
            for x in range(self.width):  
                byte_val = self.bytes[row_offset + (x // 2)]  
                if (x % 2) == 0:  
                    idx = (byte_val >> 4) & 0x0F  
      else:  
                    idx = byte_val & 0x0F  
      
      if idx < len(self.colorTable):  
                    r, g, b, _a = self.colorTable[idx]  
                else:  
                    r, g, b = 0, 0, 0 # guard against malformed palette/index  
      row_pixels.append((r, g, b))  
            grid.append(row_pixels)  
      
        self.pixelmap = grid  
        return grid
```
**`_parse_8bpp()`**  
- Every pixel is a full byte (0–255), used as an index into a 256-color palette.  
```python
   def _parse_8bpp(self):  
       if len(self.colorTable) == 0:  
           print("8bpp BMP requires a color table (palette)")  
           exit(1)  
     
       stride = self._row_stride()  
       H = self._abs_height()  
       top_down = self._is_top_down()  
     
       grid = []  
       for y in range(H):  
           src_row = y if top_down else (H - 1 - y)  
           row_offset = self.dataOffset + src_row * stride  
           row_pixels = []  
           # Each byte is an index into the palette  
     for x in range(self.width):  
               idx = self.bytes[row_offset + x]  
               if idx >= len(self.colorTable):  
                   # Guard against malformed files  
     r, g, b = 0, 0, 0  
     else:  
                   r, g, b, _a = self.colorTable[idx]  
               row_pixels.append((r, g, b))  
           grid.append(row_pixels)  
     
       self.pixelmap = grid  
       return grid
    
```

**`_parse_24bpp()`**  
- The simplest to handle, each pixel is stored directly as three bytes: **B, G, R**.  
- There’s no palette; every pixel contains its actual color.  
- Requires accounting for 4 byte row padding (each row rounded to a multiple of 4 bytes).  

```python
def _parse_24bpp(self):  
    if self.compression != 0:  
        print("24bpp parser supports only BI_RGB (no compression)")  
        exit(1)  
  
    bytes_per_pixel = 3  
  stride = self._row_stride()  # ((24*width + 31)//32)*4  
  H = self._abs_height()  
    top_down = self._is_top_down()  
  
    grid = []  
    for y in range(H):  
        src_row = y if top_down else (H - 1 - y)  
        row_offset = self.dataOffset + src_row * stride  
        row_pixels = []  
  
        payload_len = self.width * bytes_per_pixel  
        end = row_offset + payload_len  
  
        # safety clamp  
  if end > len(self.bytes):  
            raise ValueError("BMP pixel data truncated")  
  
        i = row_offset  
        for _x in range(self.width):  
            b = self.bytes[i + 0]  
            g = self.bytes[i + 1]  
            r = self.bytes[i + 2]  
            row_pixels.append((r, g, b))  
            i += 3  
  grid.append(row_pixels)  
    self.pixelmap = grid  
    return grid
```	

---

### `ImageViewer`
Displays the image and handles all transformations.

The **`rebuild()`** function is the main processing "pipeline" that I decided to stack all the scaling/rgb-masking effects onto:
1. Every re-render, it copies the raw pixel grid from the BMP.
2. While not required, for 1bpp images (that are usually dithered) I do a low pass filter using a gaussian kernel, to fix the dithering
3. Toggles color channels (R/G/B) based on button states.
4. Adjusts brightness by converting to YUV, and then adjusting the Y channel
5. Resizes the image with bilinear interpolation for smooth scaling.

```python
   def rebuild(self):  
       if not self.bmp:  
           return  
     
     src = copy.deepcopy(self.bmp.pixelmap if self.bmp.pixelmap else self.bmp.generatePixelGrid())  
     
       # Idk if this allowed, but its we do a low pass on the image with a gaussian kernel  
    # to reduce dithering artifacts  if self.bmp.bpp == 1 and self.scale < 1.0:  
           src = gaussian_blur(src, radius=1, sigma=0.8)  
     
       # 1) RGB channel toggles  
     if not (self.mask_r and self.mask_g and self.mask_b):  
           h, w = len(src), len(src[0])  
           for y in range(h):  
               row = src[y]  
               for x in range(w):  
                   r, g, b = row[x]  
                   row[x] = (  
                       r if self.mask_r else 0,  
                       g if self.mask_g else 0,  
                       b if self.mask_b else 0  
     )
      ...(rest of code here)
  ```

Finally, it writes the processed pixels back to a `QImage` and displays it. (Doesnt actually use the QImage functions as mentioned before)

---

### `bilinear_resize()`
This function enables the program to smoothly scale the image when the user moves the scale slider.
What some people did, was average the surrounding pixels only allows you to do it in discrete steps, or just used Nearest Neighbhor.
Nearest neighbhor can cause alot of aliasing since we sample below nyquist. Averaging the surrounding pixels works a bit better, but you can only do it in certain discrete steps.
After some research, I decided to go with bilinear interpolation. Its better cus you can sample any non-integer coordinate and get any scale you want.
Bilinear interpolation samples from the four nearest pixels to compute new colors, and allows any non-integer scale and produces smoother transitions when zooming in or out.
This function had some issues with dithering from the 1bpp images however. I found I could fix the artifacts by doing a low pass filter before scaling.

```python
def bilinear_resize(src, new_w, new_h, src_w=None, src_h=None, scale=None):  
    if src_h is None: src_h = len(src)  
    if src_w is None: src_w = len(src[0])  
    out = [[(0, 0, 0) for _ in range(new_w)] for _ in range(new_h)]  
  
    # map output pixel centers back to source  
 # derive scale if not provided  if scale is None:  
        scale_x = new_w / float(src_w)  
        scale_y = new_h / float(src_h)  
    else:  
        scale_x = scale_y = scale  
  
    for y_out in range(new_h):  
        src_y = (y_out + 0.5) / scale_y - 0.5  
  y0 = int(src_y)  
        y1 = min(max(y0 + 1, 0), src_h - 1)  
        wy = src_y - y0  
        if y0 < 0: y0 = 0  
  
  for x_out in range(new_w):  
            src_x = (x_out + 0.5) / scale_x - 0.5  
  x0 = int(src_x)  
            x1 = min(max(x0 + 1, 0), src_w - 1)  
            wx = src_x - x0  
            if x0 < 0: x0 = 0  
  
  c00 = src[y0][x0]  
            c10 = src[y0][x1]  
            c01 = src[y1][x0]  
            c11 = src[y1][x1]  
  
            r = int((1 - wx) * (1 - wy) * c00[0] + wx * (1 - wy) * c10[0] +  
                    (1 - wx) * wy * c01[0] + wx * wy * c11[0])  
            g = int((1 - wx) * (1 - wy) * c00[1] + wx * (1 - wy) * c10[1] +  
                    (1 - wx) * wy * c01[1] + wx * wy * c11[1])  
            b = int((1 - wx) * (1 - wy) * c00[2] + wx * (1 - wy) * c10[2] +  
                    (1 - wx) * wy * c01[2] + wx * wy * c11[2])  
  
            out[y_out][x_out] = (r, g, b)  
    return out
```

---

### `gaussian_blur()`
Idk if were allowed or supposed to do this, but it definitely applies well for reducing artifacts when scaling. Especially with 1bpp images. Alot of the artifacts are high frequency components. Convolving this kernel removes alot of it. 

It uses a 2D Gaussian kernel to average nearby pixels, giving a subtle blur that helps downsampled binary images look more natural.

```python
  
def gaussian_kernel(radius=1, sigma=1.0):  
    size = 2 * radius + 1  
  kernel = [[0.0 for _ in range(size)] for _ in range(size)]  
    sum_val = 0.0  
  for y in range(-radius, radius + 1):  
        for x in range(-radius, radius + 1):  
            val = math.exp(-(x * x + y * y) / (2.0 * sigma * sigma))  
            kernel[y + radius][x + radius] = val  
            sum_val += val  
    for j in range(size):  
        for i in range(size):  
            kernel[j][i] /= sum_val  
    return kernel  
  
  
def gaussian_blur(pixelgrid, radius=1, sigma=1.0):  
    kernel = gaussian_kernel(radius, sigma)  
    size = len(kernel)  
    h = len(pixelgrid)  
    w = len(pixelgrid[0])  
    out = [[(0, 0, 0) for _ in range(w)] for _ in range(h)]  
    for y in range(h):  
        for x in range(w):  
            r_acc = g_acc = b_acc = 0.0  
  for j in range(size):  
                for i in range(size):  
                    yy = min(max(y + j - radius, 0), h - 1)  
                    xx = min(max(x + i - radius, 0), w - 1)  
                    kr = kernel[j][i]  
                    r, g, b = pixelgrid[yy][xx]  
                    r_acc += r * kr  
                    g_acc += g * kr  
                    b_acc += b * kr  
            out[y][x] = (int(r_acc + 0.5), int(g_acc + 0.5), int(b_acc + 0.5))  
    return out
```


---

### `MainWindow`
Basically the main controller of the app. 

- Displays file info (filename, size, dimensions, bits per pixel).
- Hosts all UI components:
  - `FileDrop` area for loading BMPs
  - Scale and brightness sliders
  - RGB toggle buttons
- Connects UI events to the corresponding image functions in `ImageView`.

This ties together the parsing, image processing, and user interface into a simple interactive viewer.
```python
  
class MainWindow(QMainWindow):  
    def __init__(self):  
        super().__init__()  
        self.setWindowTitle("BMP Viewer")  
  
        self.mainwidget = QWidget()  
        layout = QVBoxLayout()  
  
        # file info  
  info_layout = QHBoxLayout()  
        self.filename_label = QLabel("Filename: ")  
        self.size_label = QLabel("Size: ")  
        self.dimensions_label = QLabel("Dimensions: ")  
        self.bpp_label = QLabel("Bits per pixel: ")  
        for w in (self.filename_label, self.size_label, self.dimensions_label, self.bpp_label):  
            info_layout.addWidget(w)  
        layout.addLayout(info_layout)
        
        ...(rest of the code here)
```


---
