# NetEq的直方图
在WebRTC中的对应实现是类Histogram

`modules/audio_coding/neteq/histogram.cc`

## fields
```cpp
private:
  std::vector<int> buckets_; 
  int forget_factor_;  // Q15
  const int base_forget_factor_;
  int add_count_;
  const absl::optional<double> start_forget_weight_;
  ```
直方图的横坐标量是IAT(Inter Arrival Time)，即数据包的到达时间间隔；纵坐标是每种IAT对应的概率，即统计时间段内对应IAT的占比。
1. buckets_ 直方图的主体，即可以将索引看作横坐标（单位规定为一个基本延迟），每个值都是一个纵坐标值。
2. forget_factor_ 遗忘因子
3. base_forget_factor_ 遗忘因子在不断的调整中要趋向的最终值
4. add_count_ 累计数

这里buckets_里存储的占比数据用Q30进行记录，这是一种使用32位整数定点表示小数的方法。

## Add

Add方法中体现了直方图是如何对每一个新数据进行统计更新的。

```cpp
void Histogram::Add(int value) {
  RTC_DCHECK(value >= 0);
  RTC_DCHECK(value < static_cast<int>(buckets_.size()));
  int vector_sum = 0; 

  for (int& bucket : buckets_) {
    bucket = (static_cast<int64_t>(bucket) * forget_factor_) >> 15;
    vector_sum += bucket;
  }

  buckets_[value] += (32768 - forget_factor_) << 15;
  vector_sum += (32768 - forget_factor_) << 15;  
  vector_sum -= 1 << 30;  
  if (vector_sum != 0) {
    int flip_sign = vector_sum > 0 ? -1 : 1;
    for (int& bucket : buckets_) {
      int correction = flip_sign * std::min(std::abs(vector_sum), bucket >> 4);
      bucket += correction;
      vector_sum += correction;
      if (std::abs(vector_sum) == 0) {
        break;
      }
    }
  }
  RTC_DCHECK(vector_sum == 0);  
  ++add_count_;
  if (start_forget_weight_) {
    if (forget_factor_ != base_forget_factor_) {
      int old_forget_factor = forget_factor_;
      int forget_factor =
          (1 << 15) * (1 - start_forget_weight_.value() / (add_count_ + 1));
      forget_factor_ =
          std::max(0, std::min(base_forget_factor_, forget_factor));
      RTC_DCHECK_GE((1 << 15) - forget_factor_,
                    ((1 << 15) - old_forget_factor) * forget_factor_ >> 15);
    }
  } else {
    forget_factor_ += (base_forget_factor_ - forget_factor_ + 3) >> 2;
  }
}
```

计算过程：
- v 新数据对应的索引
- b[i] 延迟i个单位对应的概率
- n b数组大小
- f 遗忘因子
- bf 遗忘因子极限值
- sw 遗忘因子初始权重

初步计算
$$S=\sum_0^n b[i]\times f $$

$$ b[v] = b[i] + (1-f)$$

$$S=S+(1-f) $$

调整（补偿）最后一个概率，保证归一化

$$b[n] = \left\{ \begin{array}{l} b[n]-\min(\mid S\mid , \frac{b[n]}{16}) , S > 1 \\ b[n]+\min(\mid S\mid , \frac{b[n]}{16}) , S < 1\end{array}\right.$$

更新遗忘因子，如果设置了start_forget_weight就按照下面方式更新

$$ \begin{array}{l}f=1-\frac{sw}{cnt+1} \\ f = \max(0,\min(bf, f))\end{array} $$

否则按照默认的方式

$$ f=f+\frac{(bf-f+Q_{30}(3))}{4}$$


## Quantile

```cpp
int Histogram::Quantile(int probability) {
  int inverse_probability = (1 << 30) - probability;
  size_t index = 0;      
  int sum = 1 << 30;  
  sum -= buckets_[index];

  while ((sum > inverse_probability) && (index < buckets_.size() - 1)) {
    ++index;
    sum -= buckets_[index];
  }
  return static_cast<int>(index);
}
```

根据传入的概率p取得对应的分位点B

$$ \sum^B_{i=0} b[i] > p

