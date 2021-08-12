打包和压缩
Windows的压缩文件的扩展名  .zip/.rar
linux中的打包文件：aa.tar      
linux中的压缩文件：bb.gz    
linux中打包并压缩的文件：.tar.gz

Linux中的打包文件一般是以.tar结尾的，压缩的命令一般是以.gz结尾的。
而一般情况下打包和压缩是一起进行的，打包并压缩后的文件的后缀名一般.tar.gz。

命令：tar -zcvf 打包压缩后的文件名 要打包的文件
其中：z：调用gzip压缩命令进行压缩
  c：打包文件
  v：显示运行过程
  f：指定文件名
  
示例：打包并压缩/usr/tmp 下的所有文件 压缩后的压缩包指定名称为xxx.tar
tar -zcvf ab.tar aa.txt bb.txt 
或：tar -zcvf ab.tar  *

4.2 解压
命令：tar [-zxvf] 压缩文件    
其中：x：代表解压
示例：将/usr/tmp 下的ab.tar解压到当前目录下
tar -zxvf ab.tar

示例：将/usr/tmp 下的ab.tar解压到根目录/usr下
tar -xvf ab.tar -C /usr------C代表指定解压的位置

文件的查看【查】
文件的查看命令：cat/more/less/tail

cat：看最后一屏

示例：使用cat查看/etc/sudo.conf文件，只能显示最后一屏内容
cat sudo.conf

more：百分比显示

示例：使用more查看/etc/sudo.conf文件，可以显示百分比，回车可以向下一行，空格可以向下一页，q可以退出查看
more sudo.conf

less：翻页查看

示例：使用less查看/etc/sudo.conf文件，可以使用键盘上的PgUp和PgDn向上    和向下翻页，q结束查看
less sudo.conf

tail：指定行数或者动态查看

示例：使用tail -10 查看/etc/sudo.conf文件的后10行，Ctrl+C结束  
tail -10 sudo.conf

查看某个文件第几行 # head -5 123.bak | tail -1

df -h显示磁盘使用情况
du -h显示文件系统使用情况

ln 链接 ln 123.bak 123-temp.bak
