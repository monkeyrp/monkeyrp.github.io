## JasPer简介

```JasPer项目是一个开源项目，它提供了一种基于jpeg-2000部分标准。这个项目最初是由Image Power和英属哥伦比亚大学合作完成的。目前，正在进行的JapsPer软
件的维护和开发的主要作者Michael Adams进行协调，他是维多利亚大学电子和计算机工程部门的数字信号处理组(DSPG)的成员。影响的版本是2.0.12.    
```

## 分析

异常触发

![image](C:\Users\iie.000\Desktop\图片\图1.png)

初步分析是由于函数jpc_dec.c函数中jpc_dequantize造成的core dumped

main() 到出错之间的调用关系1：

![image](C:\Users\iie.000\Desktop\图片\图2.png)

寄存器的信息1：

![image](C:\Users\iie.000\Desktop\图片\图3.jpg)

main() 到出错之间的调用关系2：

![image](C:\Users\iie.000\Desktop\图片\图4.jpg)

寄存器的信息2：

![image](C:\Users\iie.000\Desktop\图片\图5.jpg)

寄存器的信息3：

![image](C:\Users\iie.000\Desktop\图片\图6.jpg)

crash:

![image](C:\Users\iie.000\Desktop\图片\图7.jpg)

![image](C:\Users\iie.000\Desktop\图片\图8.png)

jp2_boxinfo_t *jp2_boxinfolookup(int type)遍历格式：格式

jasper的输入Jasper ./crash_in2/id_000062,sig_11,src_000901,op_ext_AO,pos_66-t jp2 

其中红色的是产生crash的文件，-T代表转换成的图像格式jpg

0×01: jasper.c

jasper.c函数241行

if (!(image = jas_image_decode(in, cmdopts->infmt,cmdopts->inopts)))

infmt是固定值4（而4 代表了jp2格式）；

而inopts来自怎么得到这个函数的来自参数解析函数，其值为0

in代表输入文件，它是由函数jas_stream_fopen()得来的；

jas_image.c 424 lines  jas_image_decode（输入文件，fmt格式，参数选项）

jas_image_lookupfmtbyid(fmt)，看fmt是否存在通过查看他的id。

对输入文件in进行解码 (*fmtinfo->ops.decode)(in,optstr)这个函数在哪定义的 ?

指针函数指向了jp2_dec.c 的97 lines，
```
dec = jp2_dec_create（）
jp2_box_get（in）在jp2_cod.c 243 lines
```
box结构，将输入文件转换为box结构
box->ops 内容赋值为0, box->ops = &jp2_boxinfo_unk.ops;
```
typedef struct {
       structjp2_boxops_s *ops;
       structjp2_boxinfo_s *info;
       uint_fast32_ttype;
       /* Thelength of the box including the (variable-length) header. */
       uint_fast32_tlen;
       /* Thelength of the box data. */
       uint_fast32_tdatalen;
       union {
              jp2_jp_t jp;
              jp2_ftyp_tftyp;
              jp2_ihdr_t ihdr;
              jp2_bpcc_tbpcc;
              jp2_colr_tcolr;
              jp2_pclr_tpclr;
              jp2_cdef_tcdef;
              jp2_cmap_tcmap;
       } data;
} jp2_box_t
```
error:
```
if (box) {
              jp2_box_destroy(box);调用次函数出错
       }
       if (dec) {
              jp2_dec_destroy(dec);
       }
       return 0;
}
void jp2_box_destroy(jp2_box_t *box) //jp2_cod.c 209lines
{
       if(box->ops->destroy) {
              (*box->ops->destroy)(box);调用指针函数出错 调用jp2_cdef_destroy
       }
       jas_free(box);
}
static void jp2_cdef_destroy(jp2_box_t *box)
{
       jp2_cdef_t*cdef = &box->data.cdef;
       if(cdef->ents) {
              jas_free(cdef->ents);
              cdef->ents= 0;
       }
}
```
解析jp2格式出错，定义box的结构一共21 ，其中两个决定了jp2格式的签名和类型，因此通过循环19次得到剩余19 个 box的数据，再获取JP2_BOX_CDEF这个box 的时候出错， jp2_dec.c 这个函数是获得输入文件，而jp2_cod.c这个函数是对每个box 进行创建和销毁。函数在调用 jas_free(cdef->ents) 的时候出错，而cdef->ents是在函数 *jp2_box_get(jas_stream_t*in) 调用
(*box->ops->getdata)(box, tmpstream)即调用jp2_cdef_getdata(jp2_box_t*box, jas_stream_t *in)函数的时候进行分配
```
static int jp2_cdef_getdata(jp2_box_t *box,jas_stream_t *in)
{
       jp2_cdef_t*cdef = &box->data.cdef;
       jp2_cdefchan_t*chan;
       unsignedint channo;
       if (jp2_getuint16(in, &cdef->numchans)) {#红色部分
              printf("numchans\n");
              return-1;
       }
       if(!(cdef->ents = jas_alloc2(cdef->numchans, sizeof(jp2_cdefchan_t)))) {
              printf("jas_alloc2mem \n");
              return-1;
       }
       printf("memhas \n");
       for(channo = 0; channo < cdef->numchans; ++channo) {
              chan= &cdef->ents[channo];
              printf("channovalue %d\n", channo);
              if(jp2_getuint16(in, &chan->channo) || jp2_getuint16(in,&chan->type) ||
                jp2_getuint16(in, &chan->assoc)) {
                     return-1;
              }
       }
       return 0;
}
```
调用上述代码红色部分，而此函数内容：
```
static int jp2_getuint16(jas_stream_t *in,uint_fast16_t *val)
{
       uint_fast16_tv;
       int c;
       if ((c = jas_stream_getc(in)) == EOF) {
              printf("jp2_getuint16c1 value:%ld\n", c);
              return-1;
       }
       v = c;
       if ((c = jas_stream_getc(in)) == EOF) {
              printf("jp2_getuint16c2 value:%d\n", c);此时 c的值为 -1
              return-1;
       }
       v = (v<< 8) | c;
       if (val) {
              printf("vvalue:%ld\n", v);
              *val= v;
       }
       return 0;
}
```
而函数jas_stream_getc(in)在stream.h中用宏定义的，而具体的宏内容是

#define  jas_stream_getc(stream)jas_stream_getc_macro(stream)使用后者替换函数，而函数jas_stream_getc_macro(stream)也是一个宏定义，其内容如下：
```
#define jas_stream_getc_macro(stream) \
       ((!((stream)->flags_& (JAS_STREAM_ERR | JAS_STREAM_EOF | \
         JAS_STREAM_RWLIMIT))) ? \
         (((stream)->rwlimit_ >= 0 && (stream)->rwcnt_ >=(stream)->rwlimit_) ? \
         (stream->flags_  |= JAS_STREAM_RWLIMIT, EOF) : \
        jas_stream_getc2(stream)): EOF)
```
flags_：0；执行黄色部分，而rwlimit_ ： -1，rwcnt_：1；由于 1>-1 执行黄色中的红色部分，执行完成或后赋值，然后flags_的值为0×0004，然后继续执行函数： jp2_getuint16(jas_stream_t*in, uint_fast16_t *val) 中的绿色语句，此时flags_的值为0×0004 ，在函数中直接执行？表达式的最后一个选项即 EOF ，然后返回。
destroy在getdata指针函数中是怎么赋值到一个数的？
datalen == 1字节
jp2_boxinfolookup()轮询box->type
copy函数是怎么工作？
jas_stream_copy()函数：
有三个参数指针out，指针in类型都是jas_stream_t ， 整型n
int all 值取决于n 此处all=0； 
## 总结
这个漏洞是指针未初始化造成的，我们采用工具[afl](http://lcamtuf.coredump.cx/afl/)挖掘出来的一个漏洞，但是当我们分析完成后，正准备提交给CVE的时候的发现，已经被别人申报了。因此，打算将我们的分析报告发送出来，请大家指正。此分析对应的漏洞CVE是[CVE-2017-6850](https://www.cvedetails.com/cve/CVE-2017-6850/)。

来自本人[freebuf](http://www.freebuf.com/vuls/138242.html)文章
