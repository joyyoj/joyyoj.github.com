
closeOp
==============

如果上一个文件结束，下一个文件开始？

If InputFileChanged || !firstFetchHappened
  set up the fetch operator for the new input file.


  
For MultiSet a,b,c

ProcessOp(tag, key, value):

If inputFileChanged:

  && firstFetchHappened:
  
// we need to first join and flush out data left by the previous file.

If isNewKey(key):
  nextKey = key; 
  foundNextKeyGroup = true;
  nextGroupStorage.add(value);
  List<Byte> smallestPos = null;
  do {
    smallestPos = joinOneGroup();
    //jump out the loop if we need input from the big table
  } while (smallestPos.size() > 0 && !smallestPos.contains((byte)this.posBigTable));
END

candidateStorage[tag].add(value);

joinOneGroup:

smallestPos = findSmallestKey();


1 a11 a12
2 a21 a22
3 a31 a32

4 a41 a42
5 a51 a52

0 b01 b02
1 b11 b12
3 b31 b32 

5 b51 b52

a b c

a = findSmallest();

