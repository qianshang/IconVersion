# IconVersion
App icon 附带版本信息

- [1. 安装imageMagick和ghostScript](#1)
- [2. 快速集成](#2)
	- [2.1. 创建脚本文件](#2.1)
	- [2.2. 集成到Xcode](#2.2)

===

## <a name="1"></a>1. 安装`imageMagick`和`ghostScript`

>`imageMagick` (TM) 是一个免费的创建、编辑、合成图片的软件。它可以读取、转换、写入多种格式的图片。
> **图片切割、颜色替换、各种效果的应用，图片的旋转、组合，文本，直线，多边形，椭圆，曲线，附加到图片伸展旋转**。

- [imageMagick官方使用文档](https://www.imagemagick.org/script/command-line-options.php)
- [imageMagic中文站](https://link.jianshu.com/?t=http://www.imagemagick.com.cn/)

使用`homebrew`安装
```
brew install imagemagick
brew install ghostscript
```

## <a name="2"></a>2. 快速集成

### <a name="2.1"></a>2.1. 创建脚本文件

1. 创建脚本文件

```
touch icon_version.sh
```

2. 修改脚本文件内容 

```
vi icon_version.sh
```
内容修改如下:
**需要将`projectName`修改为自己工程名**
```
#!/bin/sh

## Release 不执行
#echo "Configuration: $CONFIGURATION"
#if [ ${CONFIGURATION} = "Release" ]; then
#exit 0;
#fi

## 检查是否安装了ImageMagick,没有安装则不执行脚本
convertPath=`which convert`
# 判断 convertPath 文件是否存在
if [ ! -f ${convertPath}]; then
echo "==============WARNING: 你需要先安装 ImageMagick！！！！:brew install imagemagick=============="
exit 0;
fi

# 说明
# projectName   项目名称
# version       app-版本号
# build_num     app-构建版本号
# commit        git commit
# branch        git branch
projectName="IconVersion"
version=`/usr/libexec/PlistBuddy -c "Print CFBundleShortVersionString" "${CONFIGURATION_BUILD_DIR}/${INFOPLIST_PATH}"`
build_num=`/usr/libexec/PlistBuddy -c "Print CFBundleVersion" "${CONFIGURATION_BUILD_DIR}/${INFOPLIST_PATH}"`

commit=`git rev-parse --short HEAD`
branch=`git rev-parse --abbrev-ref HEAD`

shopt -s extglob
build_num="${build_num##*( )}"
shopt -u extglob
caption="${version}($build_num)\n${branch}\n${commit}"
echo $caption


function abspath() { pushd . > /dev/null; if [ -d "$1" ]; then cd "$1"; dirs -l +0; else cd "`dirname \"$1\"`"; cur_dir=`dirs -l +0`; if [ "$cur_dir" == "/" ]; then echo "$cur_dir`basename \"$1\"`"; else echo "$cur_dir/`basename \"$1\"`"; fi; fi; popd > /dev/null; }
function processIcon() {
base_file=$1
temp_path=$2
dest_path=$3
if [[ ! -e $base_file ]]; then
echo "error: file does not exist: ${base_file}"
exit -1;
fi
if [[ -z $temp_path ]]; then
echo "error: temp_path does not exist: ${temp_path}"
exit -1;
fi
if [[ -z $dest_path ]]; then
echo "error: dest_path does not exist: ${dest_path}"
exit -1;
fi
file_name=$(basename "$base_file")
final_file_path="${dest_path}/${file_name}"
base_tmp_normalizedFileName="${file_name%.*}-normalized.${file_name##*.}"
base_tmp_normalizedFilePath="${temp_path}/${base_tmp_normalizedFileName}"
# Normalize
echo "Reverting optimized PNG to normal"
echo "xcrun -sdk iphoneos pngcrush -revert-iphone-optimizations -q '${base_file}' '${base_tmp_normalizedFilePath}'"
xcrun -sdk iphoneos pngcrush -revert-iphone-optimizations -q "${base_file}" "${base_tmp_normalizedFilePath}"
width=`identify -format %w "${base_tmp_normalizedFilePath}"`
height=`identify -format %h "${base_tmp_normalizedFilePath}"`
band_height=$((($height * 50) / 100))
band_position=$(($height - $band_height))
text_position=$(($band_position))

echo "Image dimensions ($width x $height) - band height $band_height @ $band_position - point size $point_size"
#
# blur band and text
#
convert "${base_tmp_normalizedFilePath}" -blur 10x8 /tmp/blurred.png
convert /tmp/blurred.png -gamma 0 -fill white -draw "rectangle 0,$band_position,$width,$height" /tmp/mask.png
convert -size ${width}x${band_height} xc:none -fill 'rgba(0,0,0,0.2)' -draw "rectangle 0,0,$width,$band_height" /tmp/labels-base.png

point_size=$(((8 * $height) / 58))
convert -background none -size ${width}x${band_height} -pointsize $point_size -fill green -gravity center -gravity South caption:"$caption" /tmp/labels.png


convert "${base_tmp_normalizedFilePath}" /tmp/blurred.png /tmp/mask.png -composite /tmp/temp.png
rm /tmp/blurred.png
rm /tmp/mask.png
#
# compose final image
#
filename=New"${base_file}"
convert /tmp/temp.png /tmp/labels-base.png -geometry +0+$band_position -composite /tmp/labels.png -geometry +0+$text_position -geometry +${w}-${h} -composite -alpha remove "${final_file_path}"
# clean up
rm /tmp/temp.png
rm /tmp/labels-base.png
rm /tmp/labels.png
rm "${base_tmp_normalizedFilePath}"
echo "Overlayed ${final_file_path}"
}
# Process all app icons and create the corresponding internal icons
# icons_dir="${SRCROOT}/Images.xcassets/AppIcon.appiconset"
icons_path="${PROJECT_DIR}/"${projectName}"/Assets.xcassets/AppIcon.appiconset"
icons_dest_path="${PROJECT_DIR}/"${projectName}"/Assets.xcassets/AppIcon-Internal.appiconset"
icons_set=`basename "${icons_path}"`
tmp_path="${TEMP_DIR}/IconVersioning"
echo "icons_path: ${icons_path}"
echo "icons_dest_path: ${icons_dest_path}"
mkdir -p "${tmp_path}"
if [[ $icons_dest_path == "\\" ]]; then
echo "error: destination file path can't be the root directory"
exit -1;
fi
rm -rf "${icons_dest_path}"
cp -rf "${icons_path}" "${icons_dest_path}"
# Reference: https://askubuntu.com/a/343753
find "${icons_path}" -type f -name "*.png" -print0 |
while IFS= read -r -d '' file; do
echo "$file"
processIcon "${file}" "${tmp_path}" "${icons_dest_path}"
done
```

### <a name="2.2"></a>2.2. 集成到Xcode

1. 将上一步创建的脚本文件`icon_version.sh`添加到项目中
2. 配置**Xcode**中的`Build Phases`选项卡，选择`New Run Script Phase`添加`Run Script`
3. 修改`shell`内容如下
```
"${SRCROOT}/.../icon_version.sh"
```
4. 配置**Xcode**中的`General`选项卡，选择`App Icons and Launch Images`项，将`App Icons Source`修改为`AppIcon-Internal`

