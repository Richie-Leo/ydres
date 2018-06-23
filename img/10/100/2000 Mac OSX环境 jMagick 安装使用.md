jMagick：Java包装的ImageMagick接口，通过JNI实现。<br />
Mac OSX中必须通过源码编译安装jMagick。

#### 步骤：

1. 通过MacPorts源码编译安装ImageMagick。
    1. `port search ImageMagick`：从MacPorts中找到ImageMagick的最新版本；
    2. `sudo port install ImageMagick @6.9.2-0`：安装最新版ImageMagick； <br />
       备注：
       - 必须通过源码编译安装ImageMagick（从ImageMagick下载Mac压缩包，设置环境变量的方式不行），否则在编译安装jMagick时，`checking for MagickCore-config`将报错：`No package 'MagickCore’ found`。在源码编译安装过程中，会生成`MagickCore-config`所需文件。
       - 之前使用的 `port install rb-rmagick`，虽然`rb-rmagick`安装失败，但成功安装了ImageMagick的最新版本6.9.2-0。
2. 源码编译安装jMagick。
    1. 从[github.com/techblue/jmagick](https://github.com/techblue/jmagick)下载jMagick源码压缩包；
    2. 在jMagick源码目录执行： <br />
        `./configure --with-java-includes=/System/Library/Frameworks/JavaVM.framework/Headers/ --prefix=/usr/local/jmagick` <br />
        `--with-java-includes`：指定JDK头文件位置；<br />
        `--prefix`：指定jMagick安装路径为`/usr/local/jmagick`；
    3. `make all`
    4. `sudo make install `
3. Java开发、运行环境设置。
    1. `cd /usr/local/jmagick/lib/mvn` <br />
       `sudo rm -rf libJMagick.so` <br />
       `sudo mv libJMagick-6.7.7.so libJMagick-6.7.7.dylib` <br />
       `sudo ln -s libJMagick-6.7.7.dylib libJMagick.dylib`
    2.  `cd /usr/local/jmagick/lib/` <br />
        `mvn install:install-file -DgroupId=zeus -DartifactId=jmagick -Dversion=6.7.7 -Dpackaging=jar -Dfile=jmagick.jar`
    3. Eclipse运行、调试设置：<br />
       在jmagick.jar右键属性中设置Native库文件路径。
       ![image](https://github.com/Richie-Leo/ydres/blob/master/img/10/100/10-100-2000-osx-jMagic.png?raw=true)
    4. Java开发：
        - 运行时，添加JVM启动参数：`-Djava.library.path=/usr/local/jmagick/lib`
        - 在触发jmagick相关类加载前执行：`System.setProperty( "jmagick.systemclassloader" , "no" );` 或者添加JVM启动参数：`-Djmagick.systemclassloader=no`

#### 安装环境：
1. `ImageMagick`：<br />
    通过MacPorts安装的ImageMagick二进制执行文件位于 `/opt/local/bin/` 中，库文件位于 `/opt/local/lib/` 中；<br />
    ImageMagick源码手工下载存于：`/Users/richie/Documents/workspace/xcode_workspace/ImageMagick`
2. `jMagick`：<br />
    安装过程指定的安装路径为：`/usr/local/jmagick` <br />
    源码：`/Users/richie/Documents/software/jMagick`，其中包括.java、.c、.h等文件

#### jMagick使用：

```java
import magick.CompressionType;
import magick.ImageInfo;
import magick.MagickException;
import magick.MagickImage;

public class JMagickDemo {
	static {
		System.setProperty("jmagick.systemclassloader", "no");
	}

	public static void main(String[] args) throws MagickException {
		String srcFile1 = "/Users/richie/Documents/docs/research/rd-workspace/zeus-framework/basis/basis-impl/run/src-1.jpg";
		String srcFile2 = "/Users/richie/Documents/docs/research/rd-workspace/zeus-framework/basis/basis-impl/run/src-1.png";
		String dstFile1 = "/Users/richie/Documents/docs/research/rd-workspace/zeus-framework/basis/basis-impl/run/dst-1.jpg";
		String dstFile2 = "/Users/richie/Documents/docs/research/rd-workspace/zeus-framework/basis/basis-impl/run/dst-2.jpg";

		MagickImage stripped = null;

		ImageInfo info = new ImageInfo(srcFile1);
		System.out.println("=========================");
		MagickImage image = new MagickImage(info);
		printInfo(image);
		stripped = strip(image);
		resetQuality(stripped);

		stripped.setFileName(dstFile1);
		stripped.writeImage(info);
		stripped.destroyImages();
		image.destroyImages();

		info = new ImageInfo(srcFile2);
		System.out.println("=========================");

		image = new MagickImage(info);
		printInfo(image);
		image.setMagick("jpg");

		image.setCompression(CompressionType.JPEGCompression);
		stripped = strip(image);
		resetQuality(stripped);
		stripped.setFileName(dstFile2);
		stripped.writeImage(info);
		stripped.destroyImages();
		image.destroyImages();
	}

	private static void resetQuality(MagickImage image) throws MagickException {
		String format = image.getMagick();
		if ("JPEG".equals(format)) {
			int quality = image.getQuality() * 80 / 100;
			System.out.println("quality: " + image.getQuality() + " => " + quality);
			image.setQuality(quality);
		} else if ("PNG".equals(format)) {
			image.setQuality(50);
			System.out.println("quality: 0 => 50");
		}
	}

	private static MagickImage strip(MagickImage image) throws MagickException {
		// 来自ImageMagick: image.c -> StripImage(...)
		image.profileImage("*", null);
		image.setImageAttribute("comment", null);
		image.setImageAttribute("date:create", null);
		image.setImageAttribute("date:modify", null);
		// 来自网络
		image.setImageAttribute("JPEG-Sampling-factors", null);
		// 删除图片中加入的恶意信息（例如javascript代码、其他可执行代码等）
		// 使用jMagick用原图大小scale后，图片中的恶意代码会被删除掉：
		// http://blog.csdn.net/shixing_11/article/details/5720838
		return image.scaleImage(image.getDimension().width, image.getDimension().height);
	}

	private static void printInfo(MagickImage image) throws MagickException {
		System.out.println("type:" + image.getMagick());
		System.out.println("quality:" + image.getQuality());
		System.out.println("size:" + image.getDimension().width + "x" + image.getDimension().height);

		System.out.println("ISO Speed:" + image.getImageAttribute("EXIF:ISOSpeedRatings"));
		System.out.println("Flash:" + image.getImageAttribute("EXIF:Flash"));
		System.out.println("Model:" + image.getImageAttribute("EXIF:Model"));
	}
}
```