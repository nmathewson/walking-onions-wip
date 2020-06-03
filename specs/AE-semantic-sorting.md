
<!-- Section A.5 --> <a id='SA.5'></a>

## Appendix E: Semantic sorting for CBOR values.

Some voting operations assume a partial ordering on CBOR values.  We define
such an ordering as follows:

  * bstr and tstr items are sorted lexicographically, as if they were
    compared with a version of strcmp() that accepts internal NULs.
  * uint and int items are are sorted by integer values.
  * arrays are sorted lexicographically by elements.
  * Tagged items are sorted as if they were not tagged.
  * Maps do not have any sorting order.
  * False precedes true.
  * Otherwise, the ordering between two items is not defined.

More specifically:

     Algorithm: compare two cbor items A and B.

     Returns LT, EQ, GT, or NIL.

     While A is tagged, remove the tag from A.
     While B is tagged, remove the tag from B.

     If A is any integer type, and B is any integer type:
          return A cmp B

     If the type of A is not the same as the type of B:
          return NIL.

     If A and B are both booleans:
          return int(A) cmp int(B), where int(false)=0 and int(B)=1.

     If A and B are both tstr or both bstr:
          while len(A)>0 and len(B)>0:
             if A[0] != B[0]:
                  return A[0] cmp B[0]
             Discard A[0] and B[0]
          If len(A) == len(B) == 0:
             return EQ.
          else if len(A) == 0:
             return LT.  (B is longer)
          else:
             return GT.  (A is longer)

     If A and B are both arrays:
          while len(A)>0 and len(B)>0:
             Run this algorithm recursively on A[0] and B[0].
             If the result is not EQ:
                 Return that result.
             Discard A[0] and B[0]
          If len(A) == len(B) == 0:
             return EQ.
          else if len(A) == 0:
             return LT.  (B is longer)
          else:
             return GT.  (A is longer)

    Otherwise, A and B are a type for which we do not define an ordering,
    so return NIL.
