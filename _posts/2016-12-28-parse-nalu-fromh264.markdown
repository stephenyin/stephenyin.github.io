---
layout: post
title:  "从 H264 码流中解析出 nal 单元"
date:   2016-12-28 10:17:33 +0800
categories: AV
tags: AV
---
在应用层，很多时候 IDR、SEI、PPS、SPS 是打包在一起收到的， 需要手动解析出这几种帧类型。

> 定义 nalu 结构

```c
typedef struct
{
    uint32_t len;                     //! Length of the NAL unit (Excluding the start code, which does not belong to the NALU)
    int32_t forbidden_bit;            //! should be always FALSE
    int32_t nal_reference_idc;        //! NALU_PRIORITY_xxxx
    int32_t nal_unit_type;            //! NALU_TYPE_xxxx
    char *buf;                        //! contains the first byte followed by the EBSP
} NALU_t;
```

> 从 Buffer 中查找 StartCode 0x0001 或 0x001， 并返回 StartCode 的长度

```c
static int32_t FindStartCode(const char *buf)
{
    if(buf[0] != 0x0 || buf[1] != 0x0 ){
        return 0;
    }

    if(buf[2] == 0x1){
        return 3;
    } else if(buf[2] == 0x0){
        if(buf[3] == 0x1){
            return 4;
        } else {
            return 0;
        }
    } else {
        return 0;
    }
    return 0;
}
```

> 从码流中读取完整的 nalu 信息存放在 NALU_t 结构中，返回本次读取到的 Nalu 长度

```c
static int32_t ReadOneNaluFromBuf(const char *buffer, uint32_t nBufferSize, uint32_t offSet, NALU_t* pNalu)
{
    uint32_t start = 0;
    if( offSet < nBufferSize) {
        start = FindStartCode(buffer + offSet);
        if(start != 0) {
            uint32_t pos = start;
            while (offSet + pos < nBufferSize) {
                if(buffer[offSet + pos++] == 0x00 &&
                    buffer[offSet + pos++] == 0x00 &&
                    buffer[offSet + pos++] == 0x00 &&
                    buffer[offSet + pos++] == 0x01
                    ) {
                    break;
                }
            }

            if(offSet + pos == nBufferSize){
                pNalu->len = pos - start;
            } else {
                pNalu->len = (pos - 4) - start;
            }

            pNalu->buf =(char*)&buffer[offSet + start];
            pNalu->forbidden_bit = pNalu->buf[0] & 0x80;
            pNalu->nal_reference_idc = pNalu->buf[0] & 0x60; // 2 bit
            pNalu->nal_unit_type = (pNalu->buf[0]) & 0x1f;// 5 bit

            return (pNalu->len + start);
        }
    }

    return 0;
}
```

> 循环读取，直到读完 buffer 中的所有 nalu

```c
void parseH264(const char* pH264, uint32_t bufLen)
{
    NALU_t nalu;
    int32_t pos = 0, len = 0;
    len = ReadOneNaluFromBuf(pH264, bufLen, pos, &nalu);
    while (len != 0){
    	printf("## nalu type(%d)\n", nalu.nal_unit_type);
        pos += len;
        len = ReadOneNaluFromBuf(pH264, bufLen, pos, &nalu);
    }

}
```