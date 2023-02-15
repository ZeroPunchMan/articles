# ARM的原子操作指令

ARM提供了ldrex和strex指令来保证对RAM的独占操作,来保证RAM中数据操作的原子性,之前花了些时间研究了其机制,有一些经验总结.

## 背景
```
在之前的ARM架构上,提供了swp指令来实现原子操作,即通过单指令交换RAM中数据.
较新的ARM指令集,已经支持了ldrex/strex,经查阅RISC-V也是使用类似指令.
一些低成本方案(如CM0与CM0+),既没有swp指令,也没有ldrex/strex,一些异步操作的实现会比较麻烦.
```

## ldrex与strex原理
通过ARM官方文档,可以看到指令描述,大意如下:
```
1.CPU有一个内置监视器,当ldrex从RAM加载一个地址的数据到寄存器,监视器会标记这个地址.
2.strex尝试把寄存器的数据放回RAM中,如果这个地址标记存在,则成功放入并返回0;否则失败并返回1.
```

从其描述可以了解到,关键在于指令对地址的标记,使用方式如下：
```
uint8_t a;

uint8_t va = __LDREXB(&a); //ldrex加载a的值到va
 
va++; //va自增

int r = __STREXB(va, &a); //新的值放回a

if(!r) //r为0表示操作成功
{}
else //r为1表示操作失败
{}
```

另外还有一点细节,通过代码可以测试出来.

### 1.CPU只能标记1个地址
用如下代码测试,可以发现CPU同时只能标记一个地址.
```
uint8_t a, b;

uint8_t va = __LDREXB(&a); //加载a到va
va++;

uint8_t vb = __LDREXB(&b); //加载b到vb
vb++;

int rb = __STREXB(vb, &b); //把自增后的vb放回b中,会成功

int ra = __STREXB(va, &a); //把自增后的va放回a中,会失败

assert(rb == 0 && ra == 1); //返回的结果,b操作成功,a操作失败
```

### 2.中断也会清除标记
代码如下,在main中执行ldrex,然后在中断里面执行strex,会发现strex失败.
```
int main(void)
{
    uint8_t a, va;
    va = __LDREXB(&a); //加载a到va
    va++;

    //等待一次中断执行完毕

    int r = __STREXB(va, &a); //新的值放回a
    assert(r == 1); //因为产生了中断,清除了标记导致strex失败
}
```

由以上两点细节，可以得到如下注意事项：
```
1.ldrex和strex成对使用,且不能嵌套.
2.ldrex和strex之间的代码必须尽量少.
```

## 封装代码
如下封装代码,即可以对数值的任意原子操作:
```
typedef void (*OperationOnByte)(volatile uint8_t *val);

static inline void AtomicOnByte(volatile uint8_t *n, OperationOnByte op)
{
    uint8_t val;
    do
    {
        val = __LDREXB(n);
        op(&val);
    } while ((__STREXB(val, n)) != 0U);
}
```

### 互斥锁
如果操作复杂,还是用互斥锁较佳,互斥锁实现代码如下:
```
#define MUTEXT_TRY_MAX_TIMES (3) //mutex的take操作尝试次数

typedef volatile uint8_t MutexArm7m_t;

static inline void MutexArm7mInit(MutexArm7m_t *mutex)
{
    mutex[0] = 0;
}

static inline bool MutexArm7mTake(MutexArm7m_t *mutex)
{
    uint8_t tryTimes = 0;
    uint8_t val;

try_take:
    val = __LDREXB((volatile uint8_t *)mutex);
    if (val)
        return false;

    val = 1;
    if (__STREXB(val, (volatile uint8_t *)mutex))
    {
        if (++tryTimes < MUTEXT_TRY_MAX_TIMES) 
            goto try_take;

        return false;
    }
    else
    {
        return true;
    }
}

static inline void MutexArm7mGive(MutexArm7m_t *mutex)
{
    mutex[0] = 0;
}

static inline bool MutexArm7mAvailable(MutexArm7m_t *mutex)
{
    return mutex[0] == 0;
}
```



