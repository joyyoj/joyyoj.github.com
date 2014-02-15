

编译GDB ::

  LDFLAGS=-L<path to libpython2.7> ./configure --prefix=/home/users/sunshangchun/lib/gdb --with-python=/home/tools/tools/python/2.7.2/64/bin/
  make
  make install

http://sourceware.org/gdb/wiki/STLSupport

Check-out the latest Python libstdc++ printers to a place on your machine. 
In a local directory, do ::

  svn co svn://gcc.gnu.org/svn/gcc/trunk/libstdc++-v3/python

Add the following to your ~/.gdbinit. The path needs to match where the python module above was checked-out. So if checked out to: /home/maude/gdb_printers/, the path would be as written in the example ::

  python
  import sys
  sys.path.insert(0, '/home/users/sunshangchun/gdb_printers/python')
  from libstdcxx.v6.printers import register_libstdcxx_printers
  register_libstdcxx_printers (None)
  end

