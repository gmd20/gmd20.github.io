```text
//- --------------------linux 内核里面的memcpy汇编----------------
static __always_inline void *__memcpy(void *to, const void *from, size_t n)
{
        int d0, d1, d2;
        asm volatile("rep ; movsl\n\t"
                     "movl %4,%%ecx\n\t"
                     "andl $3,%%ecx\n\t"
                     "jz 1f\n\t"
                     "rep ; movsb\n\t"
                     "1:"
                     : "=&c" (d0), "=&D" (d1), "=&S" (d2)
                     : "0" (n / 4), "g" (n), "1" ((long)to), "2" ((long)from)
                     : "memory");
        return to;
}

//-----------改写成一个vc版本的内联汇编----------------------
static inline void *__memcpy(void *to, const void *from, size_t n)
{
    __asm {
        mov	ecx, n
        shr ecx, 2
        mov ebx, ecx		
        mov edi, to
        mov esi, from
        rep movs dword ptr [edi], dword ptr [esi]
        mov ecx, n
        and ecx, ebx
        jz END
        rep movs byte ptr [edi], byte ptr [esi]
    END:
	}
    return to;
}

//-----------------看看下面这个代码-------------------------
 char aaaa[256];
 
 {
 QueryPerformanceCounter(&t0);
 int i, j;
 for (i=0; i< 500000; i++) {
	 char * a = new char[256];
	 __memcpy(aaaa,a,256);
	 delete [] a;
 }
 QueryPerformanceCounter(&t1);
 unsigned long time = (((t1.QuadPart-t0.QuadPart)*1000000)/freq.QuadPart);
 std::cout  << "执行 " << i <<" 次， 耗时 " << time <<  " 微秒" << std::endl;
 }

 {
 QueryPerformanceCounter(&t0);
 int i, j;
 for (i=0; i< 500000; i++) {
	 char * a = new char[256];
	 memcpy(aaaa,a,256);
	 delete [] a;
 }
 QueryPerformanceCounter(&t1);
 unsigned long time = (((t1.QuadPart-t0.QuadPart)*1000000)/freq.QuadPart);
 std::cout  << "执行 " << i <<" 次， 耗时 " << time <<  " 微秒" << std::endl;
 }

 std::cout << aaaa[0];


//-----------------------------------



for (i=0; i< 500000; i++) {
	 char * a = new char[256];
00401900  push        100h  
00401905  call        operator new[] (40AEBAh)  
0040190A  add         esp,4  
	 memcpy(aaaa,a,256);
0040190D  mov         esi,eax  
0040190F  mov         ecx,40h  
00401914  lea         edi,[ebp-1E4h]  
	 delete [] a;
0040191A  push        eax  
0040191B  rep movs    dword ptr es:[edi],dword ptr [esi]               // 使用系统的memcpy函数，因为复制的长度知道，这里已经被优化成一条指令了  
0040191D  call        operator delete[] (40CA08h)  
00401922  add         esp,4  
00401925  dec         ebx  
00401926  jne         main+150h (401900h)  
 }

//-----------------------------------
004019FA  lea         edx,[ebp-18h]  
004019FD  push        edx  
004019FE  call        ebx  
	 __memcpy(aaaa,a,256);
00401A00  lea         eax,[ebp-1E4h]  
00401A06  mov         dword ptr [ebp-1Ch],eax  
00401A09  mov         dword ptr [ebp-10h],7A120h  
	 char * a = new char[256];
00401A10  push        100h  
00401A15  call        operator new[] (40AEBAh)  
00401A1A  add         esp,4  
00401A1D  mov         dword ptr [ebp-3Ch],eax  
	 __memcpy(aaaa,a,256);                 //自己的写的汇编是不会优化的，但内联过来了。
00401A20  mov         ecx,100h  
00401A25  shr         ecx,2  
00401A28  mov         ebx,ecx  
00401A2A  mov         edi,dword ptr [ebp-1Ch]  
00401A2D  mov         esi,dword ptr [ebp-3Ch]  
00401A30  rep movs    dword ptr es:[edi],dword ptr [esi]  
00401A32  mov         ecx,100h  
00401A37  and         ecx,ebx  
00401A39  je          main+28Dh (401A3Dh)  
00401A3B  rep movs    byte ptr es:[edi],byte ptr [esi]  
	 delete [] a;
00401A3D  push        eax  
00401A3E  call        operator delete[] (40CA08h)  
00401A43  add         esp,4  
00401A46  dec         dword ptr [ebp-10h]  
00401A49  jne         main+260h (401A10h)  
 }


//---------------再改个测试，然他不知道复制的长度，不能优化了------------------

 char aaaa[256];
   {
 QueryPerformanceCounter(&t0);
 int i, j;
 for (i=0; i< 500000; i++) {
	 char * a = new char[256];
	 memcpy(aaaa,a, i%256);
	 delete [] a;
 }
 QueryPerformanceCounter(&t1);
 unsigned long time = (((t1.QuadPart-t0.QuadPart)*1000000)/freq.QuadPart);
 std::cout  << "执行 " << i <<" 次， 耗时 " << time <<  " 微秒" << std::endl;
 }



 {
 QueryPerformanceCounter(&t0);
 int i, j;
 for (i=0; i< 500000; i++) {
	 char * a = new char[256];
	 __memcpy(aaaa,a, i%156);
	 delete [] a;
 }
 QueryPerformanceCounter(&t1);
 unsigned long time = (((t1.QuadPart-t0.QuadPart)*1000000)/freq.QuadPart);
 std::cout  << "执行 " << i <<" 次， 耗时 " << time <<  " 微秒" << std::endl;
 }


 std::cout << aaaa[0];

//----------------------------------

	 memcpy(aaaa,a, i%256);
0040188C  movzx       eax,bl  
0040188F  add         esp,4  
00401892  push        eax  
00401893  lea         ecx,[ebp-110h]  
00401899  push        esi  
0040189A  push        ecx  
0040189B  call        memcpy (40D7F0h)          //调用系统的memcpy，没有内联过来，
004018A0  add         esp,0Ch  
	 delete [] a;



这个memcpy 在 memcpy.asm文件里面定义

//--------------------------------------------------------------------------------

;Entry:
;       void *dst = pointer to destination buffer
;       const void *src = pointer to source buffer
;       size_t count = number of bytes to copy
;
;Exit:
;       Returns a pointer to the destination buffer in AX/DX:AX
;
;Uses:
;       CX, DX
;
;Exceptions:
;*******************************************************************************

ifdef MEM_MOVE
        _MEM_     equ <memmove>
else  ; MEM_MOVE
        _MEM_     equ <memcpy>
endif  ; MEM_MOVE

%       public  _MEM_
_MEM_   proc \
        dst:ptr byte, \
        src:ptr byte, \
        count:IWORD

              ; destination pointer
              ; source pointer
              ; number of bytes to copy

;       push    ebp             ;U - save old frame pointer
;       mov     ebp, esp        ;V - set new frame pointer

        push    edi             ;U - save edi
        push    esi             ;V - save esi

        mov     esi,[src]       ;U - esi = source
        mov     ecx,[count]     ;V - ecx = number of bytes to move

        mov     edi,[dst]       ;U - edi = dest

;
; Check for overlapping buffers:
;       If (dst <= src) Or (dst >= src + Count) Then
;               Do normal (Upwards) Copy
;       Else
;               Do Downwards Copy to avoid propagation
;

        mov     eax,ecx         ;V - eax = byte count...

        mov     edx,ecx         ;U - edx = byte count...
        add     eax,esi         ;V - eax = point past source end

        cmp     edi,esi         ;U - dst <= src ?
        jbe     short CopyUp    ;V - yes, copy toward higher addresses

        cmp     edi,eax         ;U - dst < (src + count) ?
        jb      CopyDown        ;V - yes, copy toward lower addresses

;
; Copy toward higher addresses.
;
CopyUp:
;
; First, see if we can use a "fast" copy SSE2 routine
        ; block size greater than min threshold?
        cmp     ecx,080h
        jb      Dword_align
        ; SSE2 supported?
        cmp     DWORD PTR __sse2_available,0
        je      Dword_align
        ; alignments equal?
        push    edi
        push    esi
        and     edi,15
        and     esi,15
        cmp     edi,esi
        pop     esi
        pop     edi
        jne     Dword_align

        ; do fast SSE2 copy, params already set
        jmp     _VEC_memcpy
        ; no return
;
; The algorithm for forward moves is to align the destination to a dword
; boundary and so we can move dwords with an aligned destination.  This
; occurs in 3 steps.
;
;   - move x = ((4 - Dest & 3) & 3) bytes
;   - move y = ((L-x) >> 2) dwords
;   - move (L - x - y*4) bytes
;

Dword_align:
        test    edi,11b         ;U - destination dword aligned?
        jnz     short CopyLeadUp ;V - if we are not dword aligned already, align

        shr     ecx,2           ;U - shift down to dword count
        and     edx,11b         ;V - trailing byte count

        cmp     ecx,8           ;U - test if small enough for unwind copy
        jb      short CopyUnwindUp ;V - if so, then jump

        rep     movsd           ;N - move all of our dwords

        jmp     dword ptr TrailUpVec[edx*4] ;N - process trailing bytes

;
; Code to do optimal memory copies for non-dword-aligned destinations.
;

; The following length check is done for two reasons:
;
;    1. to ensure that the actual move length is greater than any possiale
;       alignment move, and
;
;    2. to skip the multiple move logic for small moves where it would
;       be faster to move the bytes with one instruction.
;

        align   @WordSize
CopyLeadUp:

        mov     eax,edi         ;U - get destination offset
        mov     edx,11b         ;V - prepare for mask

        sub     ecx,4           ;U - check for really short string - sub for adjust
        jb      short ByteCopyUp ;V - branch to just copy bytes

        and     eax,11b         ;U - get offset within first dword
        add     ecx,eax         ;V - update size after leading bytes copied

        jmp     dword ptr LeadUpVec[eax*4-4] ;N - process leading bytes

        align   @WordSize
ByteCopyUp:
        jmp     dword ptr TrailUpVec[ecx*4+16] ;N - process just bytes

        align   @WordSize
CopyUnwindUp:
        jmp     dword ptr UnwindUpVec[ecx*4] ;N - unwind dword copy

        align   @WordSize
LeadUpVec       dd      LeadUp1, LeadUp2, LeadUp3

        align   @WordSize
LeadUp1:
        and     edx,ecx         ;U - trailing byte count
        mov     al,[esi]        ;V - get first byte from source

        mov     [edi],al        ;U - write second byte to destination
        mov     al,[esi+1]      ;V - get second byte from source

        mov     [edi+1],al      ;U - write second byte to destination
        mov     al,[esi+2]      ;V - get third byte from source

        shr     ecx,2           ;U - shift down to dword count
        mov     [edi+2],al      ;V - write third byte to destination

        add     esi,3           ;U - advance source pointer
        add     edi,3           ;V - advance destination pointer

        cmp     ecx,8           ;U - test if small enough for unwind copy
        jb      short CopyUnwindUp ;V - if so, then jump

        rep     movsd           ;N - move all of our dwords

        jmp     dword ptr TrailUpVec[edx*4] ;N - process trailing bytes

        align   @WordSize
LeadUp2:
        and     edx,ecx         ;U - trailing byte count
        mov     al,[esi]        ;V - get first byte from source

        mov     [edi],al        ;U - write second byte to destination
        mov     al,[esi+1]      ;V - get second byte from source

        shr     ecx,2           ;U - shift down to dword count
        mov     [edi+1],al      ;V - write second byte to destination

        add     esi,2           ;U - advance source pointer
        add     edi,2           ;V - advance destination pointer

        cmp     ecx,8           ;U - test if small enough for unwind copy
        jb      short CopyUnwindUp ;V - if so, then jump

        rep     movsd           ;N - move all of our dwords

        jmp     dword ptr TrailUpVec[edx*4] ;N - process trailing bytes

        align   @WordSize
LeadUp3:
        and     edx,ecx         ;U - trailing byte count
        mov     al,[esi]        ;V - get first byte from source

        mov     [edi],al        ;U - write second byte to destination
        add     esi,1           ;V - advance source pointer

        shr     ecx,2           ;U - shift down to dword count
        add     edi,1           ;V - advance destination pointer

        cmp     ecx,8           ;U - test if small enough for unwind copy
        jb      short CopyUnwindUp ;V - if so, then jump

        rep     movsd           ;N - move all of our dwords

        jmp     dword ptr TrailUpVec[edx*4] ;N - process trailing bytes

        align   @WordSize
UnwindUpVec     dd      UnwindUp0, UnwindUp1, UnwindUp2, UnwindUp3
                dd      UnwindUp4, UnwindUp5, UnwindUp6, UnwindUp7

UnwindUp7:
        mov     eax,[esi+ecx*4-28] ;U - get dword from source
                                   ;V - spare
        mov     [edi+ecx*4-28],eax ;U - put dword into destination
UnwindUp6:
        mov     eax,[esi+ecx*4-24] ;U(entry)/V(not) - get dword from source
                                   ;V(entry) - spare
        mov     [edi+ecx*4-24],eax ;U - put dword into destination
UnwindUp5:
        mov     eax,[esi+ecx*4-20] ;U(entry)/V(not) - get dword from source
                                   ;V(entry) - spare
        mov     [edi+ecx*4-20],eax ;U - put dword into destination
UnwindUp4:
        mov     eax,[esi+ecx*4-16] ;U(entry)/V(not) - get dword from source
                                   ;V(entry) - spare
        mov     [edi+ecx*4-16],eax ;U - put dword into destination
UnwindUp3:
        mov     eax,[esi+ecx*4-12] ;U(entry)/V(not) - get dword from source
                                   ;V(entry) - spare
        mov     [edi+ecx*4-12],eax ;U - put dword into destination
UnwindUp2:
        mov     eax,[esi+ecx*4-8] ;U(entry)/V(not) - get dword from source
                                  ;V(entry) - spare
        mov     [edi+ecx*4-8],eax ;U - put dword into destination
UnwindUp1:
        mov     eax,[esi+ecx*4-4] ;U(entry)/V(not) - get dword from source
                                  ;V(entry) - spare
        mov     [edi+ecx*4-4],eax ;U - put dword into destination

        lea     eax,[ecx*4]     ;V - compute update for pointer

        add     esi,eax         ;U - update source pointer
        add     edi,eax         ;V - update destination pointer
UnwindUp0:
        jmp     dword ptr TrailUpVec[edx*4] ;N - process trailing bytes

;-----------------------------------------------------------------------------

// 可以看到这个函数非常的长，然后会检查长度之类的。在我机器不支持SSE2，直接跳到Dword_align: 那里开始执行。

执行 500000 次， 耗时 84557 微秒
执行 500000 次， 耗时 81347 微秒

整个循环跑下来，可以看到系统内部实现的memcpy和  自己转换的linux内核里面的memcpy内联函数是差不多的，自己的内联函数稍微快那么一点点。

综上所述，如果memcpy复制的长度已经知道的情况下，使用系统的memcpy函数，编译器可以优化的非常的好， 可以等价于一两条 rep movs汇编指令，这种情况下自己的内联memcpy不会优化，极限自己的memcpy会慢很多（长度不是4的倍数的时候）

如果memcpy的复制长度运行时才行确定，系统默认的memcpy有sse2指令的检查。如果没有sse2可用的情况下也是和自己的内联汇编几乎一样的快。不过因为系统默认memcpy函数不是内联的，实际应用中可能跳到函数代码时可能有影响 代码段的cache，有可能cache不命中的情况下会慢一些？ 那就不如自己内联的memcpy好。不过这仅仅是猜测而已，很难说这里的代码跳转就会导致代码cache的不命中。

```
