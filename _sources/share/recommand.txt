
vim支持汉字
-------------------

::

  ./configure --with-features=big
  ./configure --prefix=/usr --enable-multibyte
  vim --version
  looking for +multi_byte. If it says -multi_byte it will not work.


安装bundle
-----------
::
   git clone https://github.com/gmarik/vundle.git ~/.vim/bundle/vundle
