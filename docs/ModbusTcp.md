## Summary
本篇文章主要讲解如何去读写支持modbus-Tcp协议的设备，通常针对的是线圈和寄存器数据，本类库支持错误信息的回复，
使用本类库，数据交互将会变得非常简单。

This article mainly explains how to read and write devices that support the modbus-Tcp protocol,
which is usually aimed at the coil and register data. This library supports the recovery of error information.
Using this library, data interaction will become very simple.

## Contact
QQ Group : [592132877](http://shang.qq.com/wpa/qunwpa?idkey=2278cb9c2e0c04fc305c43e41acff940499a34007dfca9e83a7291e726f9c4e8)

Email: hsl200909@163.com

## Prepare
在测试的时候，通常需要个服务器，如果你没有真实的设备，可以使用 **ModbusTcpServer** 项目所生成的测试服务器。

When testing, you usually need a server. If you do not have a real device, you can use the test server generated by the **ModbusTcpServer** project.


## Instantiation
**1. Add namespace**

```
using HslCommunication.Modbus;
using HslCommunication;

```

**2. Declare**
```
private ModbusTcpNet busTcpClient = null;
```


**3. Instantiation**

```

// specify ip address, port, modbus station
busTcpClient = new busTcpClient("192.168.0.100",502,0x02);
busTcpClient.ConnectServer();

```

If we want to know whether connectd, we can do like this

```

OperateResult connect = busTcpClient.ConnectServer( );
if (connect.IsSuccess)
{
    // success
}
else
{
    // failed
}

```

**4. Close the connection when the program exits**

```

 busTcpClient.ConnectClose( );

```

## Exchange Data

**1. Read Or Write Coil Data**
```
OperateResult<bool[]> read = busTcpClient.ReadCoil( "100", 10 );
if(read.IsSuccess)
{
    bool coil_100 = read.Content[0];
    // and so on 
    bool coil_109 = read.Content[9];
}
else
{
    // failed
    string err = read.Message;
}
```

```
bool[] values = new bool[] { true, false, false, false, true, true, false, true, false, false };
OperateResult write = busTcpClient.WriteCoil( "100", values );
if (write.IsSuccess)
{
    // success
}
else
{
    // failed
    string err = write.Message;
}
```

**2. Read Register Include Many Data Types**
```
short m100_short = busTcpClient.ReadInt16( "100" ).Content;
ushort m100_ushort = busTcpClient.ReadUInt16( "100" ).Content;
int m100_int = busTcpClient.ReadInt32( "100" ).Content;
uint m100_uint = busTcpClient.ReadUInt32( "100" ).Content;
float m100_float = busTcpClient.ReadFloat( "100" ).Content;
double m100_double = busTcpClient.ReadDouble( "100" ).Content;
// need to specify the text length
string m100_string = busTcpClient.ReadString( "100", 10 ).Content;
```

**3. Write Register Include Many Data Types**
```
busTcpClient.Write( "100", (short)5 );
busTcpClient.Write( "100", (ushort)5 );
busTcpClient.Write( "100", 5 );
busTcpClient.Write( "100", (uint)5 );
busTcpClient.Write( "100", (long)5 );
busTcpClient.Write( "100", (ulong)5 );
busTcpClient.Write( "100", 5f );
busTcpClient.Write( "100", 5d );
busTcpClient.Write( "100", "12345678" );
```

> This library alse support write array values.


**4. Read complex data, for example, 100-109 contains all data you want**

Data name | Data section | Data type | Data Length
-|-|-|-
count | 100-101 | int | 4-byte
temp | 102-103 | float | 4-byte
name1 | 104-105 | short | 2-byte
barcode | 106-109 | string | 10-byte

So we can do like this

```

OperateResult<byte[]> read = busTcpClient.Read( "100", 10 );
if(read.IsSuccess)
{
    int count = busTcpClient.ByteTransform.TransInt32( read.Content, 0 );
    float temp = busTcpClient.ByteTransform.TransSingle( read.Content, 4 );
    short name1 = busTcpClient.ByteTransform.TransInt16( read.Content, 8 );
    string barcode = Encoding.ASCII.GetString( read.Content, 10, 10 );
}


```

**5. Implementing custom type reads and writes**

We found the code above is awkward and we want to improve.

First, Inherit and implement interface methods

```

public class UserType : HslCommunication.IDataTransfer
{
    #region IDataTransfer

    private HslCommunication.Core.IByteTransform ByteTransform = new HslCommunication.Core.ReverseWordTransform();


    public ushort ReadCount => 10;

    public void ParseSource( byte[] Content )
    {
        int count = ByteTransform.TransInt32( Content, 0 );
        float temp = ByteTransform.TransSingle( Content, 4 );
        short name1 = ByteTransform.TransInt16( Content, 8 );
        string barcode = Encoding.ASCII.GetString( Content, 10, 10 );
    }

    public byte[] ToSource( )
    {
        byte[] buffer = new byte[20];
        ByteTransform.TransByte( count ).CopyTo( buffer, 0 );
        ByteTransform.TransByte( temp ).CopyTo( buffer, 4 );
        ByteTransform.TransByte( name1 ).CopyTo( buffer, 8 );
        Encoding.ASCII.GetBytes( barcode ).CopyTo( buffer, 10 );
        return buffer;
    }


    #endregion


    #region Public Data

    public int count { get; set; }

    public float temp { get; set; }

    public short name1 { get; set; }

    public string barcode { get; set; }

    #endregion
}

```

So we can do like this

```

OperateResult<UserType> read = busTcpClient.ReadCustomer<UserType>( "100" );
if (read.IsSuccess)
{
    UserType value = read.Content;
}
// write value
busTcpClient.WriteCustomer( "100", new UserType( ) );

```

## Note

在读取线圈数据的时候，最大长度为2040，在读取寄存器数据的时候，最大长度没有限制，因为内部已经自动分割并重组了。

When reading the coil data, the maximum length is 2040. When reading the register data, the maximum length is not limited, 
because the internal has been automatically split and reorganized.


## Log Support

This component can also achieve log output. Here is an example. The specific log function refers to the logbook.

```
busTcpClient.LogNet = new HslCommunication.LogNet.LogNetSingle( Application.StartupPath + "\\Logs.txt" );
```



## Others
For more details, you can download source code, refer to HslCommunicationDemo project