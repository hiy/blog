---
author: hiy
datetime: 2019-11-01
title: レーザーハープをつくってみる
slug:
featured: false
draft: true
tags:
  - ESP32
  - mruby_c
  - SonicPi
ogImage: ""
description: レーザーを使った楽器で、レーザーを遮ることで音を出します。

---
# レーザーハープとは？
レーザーを使った楽器で、レーザーを遮ることで音を出します。
"レーザーハープ"でググると色々な例を見れますし、自作しておられる方もいらっしゃいます。

[レーザーハープをGoogle検索](https://www.google.com/search?q=%E3%83%AC%E3%83%BC%E3%82%B6%E3%83%BC%E3%83%8F%E3%83%BC%E3%83%97)

興味があったので、ESP32とmruby/cでつくってみる内容を記録します。
開発環境はmacOSを使用しています。

[ソースコード](https://github.com/hiy/laser-harp)

# 開発環境の構築

**参考資料**
[itocさんの ESP32×mruby／c IoTハンズオン](https://www.s-itoc.jp/activity/research/mrubyc/mrubyc_tutorial/ESP32/)

開発環境の構築に、上記の内容を利用させていただきました。
ビルド・実行は参考資料の環境で行います。

# 材料

| 材料名                                                                     | 数量 | 備考                                               |
| -------------------------------------------------------------------------- | ---- | -------------------------------------------------- |
| [ＥＳＰ３２－ＤｅｖＫｉｔＣ](http://akizukidenshi.com/catalog/g/gM-11819/) | 1    |                                                    |
| [フォトトランジスタ](http://akizukidenshi.com/catalog/g/gI-02325/)         | 2    | 照度センサです。光を当てると大きな電流が流れます。 |
| [レーザー発光モジュール](http://akizukidenshi.com/catalog/g/gM-00765/)     | 2    |                                                    |
| ブレッドボード                                                             | 1    |                                                    |
| 抵抗10kΩ                                                                   | 2    |                                                    |
| 洗濯バサミ                                                                 | 適量 | レーザーを固定するのに使用しました。               |


# 構成

レーザーが遮られたらWi-Fi経由でメッセージをPCに送信します。PCはメッセージを受けとって音を鳴らします。音の再生にはSonic Piを使用します。
[Sonic Pi](https://sonic-pi.net/)はRubyのコードで音楽を作成して演奏できるツールです。
Sonic Piでは、OSC(Open Sound Control　後述)という形式のメッセージを受け取って音を鳴らすことができます。

![f9a681e5-cb93-b997-1af1-681c7908be9d](https://user-images.githubusercontent.com/2444124/208229100-aaf69dd8-5fab-4d70-b671-ac15334b25ed.png)



# レーザー側（ＥＳＰ３２）の実装

レーザーを遮り、メッセージをPCに送信する処理を実装する以下のような回路を作成しました。
緑色のLEDの部分がフォトトランジスタです。


![laser-harp-003.png](https://user-images.githubusercontent.com/2444124/208228868-84a003e9-adec-44f4-a6f5-386de01ccb1c.png)

※電子工作初心者のため、おかしなところがあるかもしれません。

## 回路の説明

ESP32のADC(analog to digital converter)とフォトトランジスタを繋ぎます。
後述するプログラムで、GPI34,GPI35に入力される明るさの値を測定します。

#### 1つ目のレーザーモジュール用のフォトトランジスタ
 GPI34、GPIO32、GNDに繋ぎます。
#### 2つ目のレーザーモジュール用のフォトトランジスタ
 GPI35、GPIO33、GNDに繋ぎます。

注意点
  * フォトトランジスタの短い方の足をGPI34,35に繋ぎます。
  * ESP32のADCにはADC1とADC2があります。ADC2を使う場合、現時点ではWi-Fiと一緒に使えないようなので注意が必要です。図の回路ではADC1を使用しています。

回路については以上です。

## プログラム

次にESP32に書き込むプログラムを実装します。
実装には[mruby/c](https://github.com/mrubyc/mrubyc)を使用します。
※ mruby/cからESP32の開発環境[esp-idf](https://github.com/espressif/esp-idf)を利用します。

### 1. Wi-Fiとの接続　
Wi-Fiとの接続処理を実装します。
c_set_up_wifi関数を、mruby/cのsetup_wifiメソッドとして定義します。
SSIDとパスワード、PCのIPは環境によって書き換えが必要です。
メッセージ送信先のポート番号（Sonic Piの待受ポート）は4559固定です。

※ 実装には[esp-idfテンプレート](https://github.com/espressif/esp-idf-template)を参考にしました。

esp32/main/main.c

```c
#define WIFI_SSID "YOUR_WIFI_SSID"
#define WIFI_PASSWORD "YOUR_WIFI_PASSWORD"
#define HOST_IP_ADDRESS "192.168.0.Y"
#define HOST_OSC_PORT 4559

(中略)

static void c_setup_wifi(mrb_vm *vm, mrb_value *v, int argc)
{
  char addr_str[128];
  int addr_family;
  int ip_protocol;

  nvs_flash_init();
  tcpip_adapter_init();
  ESP_ERROR_CHECK(esp_event_loop_init(ESP_OK, NULL));
  wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
  ESP_ERROR_CHECK(esp_wifi_init(&cfg));
  ESP_ERROR_CHECK(esp_wifi_set_storage(WIFI_STORAGE_RAM));
  ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_STA));

  wifi_config_t sta_config = {
      .sta = {
          .ssid = WIFI_SSID,
          .password = WIFI_PASSWORD,
          .bssid_set = false}};
  ESP_ERROR_CHECK(esp_wifi_set_config(WIFI_IF_STA, &sta_config));
  ESP_ERROR_CHECK(esp_wifi_start());
  ESP_ERROR_CHECK(esp_wifi_connect());

  while (1)
  {
    dest_addr.sin_addr.s_addr = inet_addr(HOST_IP_ADDRESS);
    dest_addr.sin_family = AF_INET;
    dest_addr.sin_port = htons(HOST_OSC_PORT);
    addr_family = AF_INET;
    ip_protocol = IPPROTO_IP;
    inet_ntoa_r(dest_addr.sin_addr, addr_str, sizeof(addr_str) - 1);

    sock = socket(addr_family, SOCK_DGRAM, ip_protocol);
    if (sock < 0)
    {
      ESP_LOGE(TAG, "Unable to create socket: errno %d", errno);
    }
    else
    {
      ESP_LOGI(TAG, "Socket created, sending to %s:%d:%d", HOST_IP_ADDRESS, HOST_OSC_PORT, sock);
      break;
    }

    if (sock != -1)
    {
      ESP_LOGE(TAG, "Shutting down socket and restarting...");
      shutdown(sock, 0);
      close(sock);
    }
  }
}

(中略)

void app_main(void)
{
  .....
  mrbc_define_method(0, mrbc_class_object, "setup_wifi", c_setup_wifi);
  .....
}
```

esp32/mrblib/models/wifi.rb

```ruby
class Wifi
  def self.setup
    setup_wifi
  end
end
```


### 2. フォトトランジスタの明るさを測る

フォトトランジスタで明るさを測定する処理を実装します。

実装にはesp-idfの[ADC1 Example](https://github.com/espressif/esp-idf/tree/master/examples/peripherals/adc)を参考にしました。

esp32/main/main.c

```c

static esp_adc_cal_characteristics_t *adc_chars;
static const adc_channel_t channel_1 = ADC_CHANNEL_6; //GPIO34
static const adc_channel_t channel_2 = ADC_CHANNEL_7; //GPIO35
static const adc_atten_t atten = ADC_ATTEN_DB_11;
static const adc_unit_t unit = ADC_UNIT_1;

（中略）

static void c_gpio_init_output(mrb_vm *vm, mrb_value *v, int argc)
{
  int pin = GET_INT_ARG(1);
  console_printf("init pin %d\n", pin);
  gpio_set_direction(pin, GPIO_MODE_OUTPUT);
}

static void c_gpio_set_level(mrb_vm *vm, mrb_value *v, int argc)
{
  int pin = GET_INT_ARG(1);
  int level = GET_INT_ARG(2);
  gpio_set_level(pin, level);
}

static void c_init_adc(mrb_vm *vm, mrb_value *v, int argc)
{
  int pin = GET_INT_ARG(1);
  adc1_config_width(ADC_WIDTH_BIT_12);
  if (pin == 34)
  {
    adc1_config_channel_atten(channel_1, atten);
  }
  else if (pin == 35)
  {
    adc1_config_channel_atten(channel_2, atten);
  };
  adc_chars = calloc(1, sizeof(esp_adc_cal_characteristics_t));
  esp_adc_cal_value_t val_type = esp_adc_cal_characterize(unit, atten, ADC_WIDTH_BIT_12, DEFAULT_VREF, adc_chars);
}

static void c_read_adc(mrb_vm *vm, mrb_value *v, int argc)
{
  int pin = GET_INT_ARG(1);
  uint32_t adc_reading = 0;
  for (int i = 0; i < NO_OF_SAMPLES; i++)
  {
    if (pin == 34)
    {
      adc_reading += adc1_get_raw((adc1_channel_t)channel_1);
    }
    else if (pin == 35)
    {
      adc_reading += adc1_get_raw((adc1_channel_t)channel_2);
    }
  }
  adc_reading /= NO_OF_SAMPLES;
  uint32_t millivolts = esp_adc_cal_raw_to_voltage(adc_reading, adc_chars);
  SET_INT_RETURN(millivolts);
}

void app_main(void)
{
  .....

  mrbc_define_method(0, mrbc_class_object, "gpio_init_output", c_gpio_init_output);
  mrbc_define_method(0, mrbc_class_object, "gpio_set_level", c_gpio_set_level);
  mrbc_define_method(0, mrbc_class_object, "init_adc", c_init_adc);
  mrbc_define_method(0, mrbc_class_object, "read_adc", c_read_adc);

  .....
}
```

esp32/mrblib/models/phototransistor.rb

```ruby
class Phototransistor
  def initialize(gpio_pin, adc_pin)
    gpio_init_output(gpio_pin)
    gpio_set_level(gpio_pin, 1)
    init_adc(adc_pin)
    @adc_pin = adc_pin
  end

  # valueメソッドを実行することで、adcの値を読みます。
  def value
    read_adc(@adc_pin)
  end
end
```

### 3. レーザーが遮られた時にOSCメッセージ送信

次にOSCのメッセージを送信する処理を実装します。

OSCについては、wikipediaの記述のとおりです。
  > OpenSound Control（OSC）とは、電子楽器（特にシンセサイザー）やコンピュータなどの機器において音楽演奏データをネットワーク経由でリアルタイムに共有するための通信プロトコルである。

　　wikipediaの[OpenSound_Controlのページ](https://ja.wikipedia.org/wiki/OpenSound_Control)

今回は、このプロトコルのメッセージをESP32で送信します。

OSCのメッセージは、URLに似た形式です。詳しい仕様は[こちら](http://opensoundcontrol.org/spec-1_0)。
今回は、以下のメッセージを送信します。（※データをつけて送ることもできます）

<pre>
# 送信するメッセージ

# 1つ目のレーザーが遮られた時に送信するメッセージ
/trigger/laser_1

# 2つ目のレーザーが遮られた時に送信するメッセージ
/trigger/laser_2
</pre>

次に、OSCの仕様に従ってメッセージをエンコードします。
エンコードは以下のRubyのgemで行いました。

<pre>
# osc-rubyをインストール
$ gem install osc-ruby
</pre>

encode.rb

```ruby
require 'osc-ruby'
message = OSC::Message.new('/trigger/laser_1')
message.encode
# エンコードされたメッセージ
# => "/trigger/laser_1\x00\x00\x00\x00,\x00\x00\x00"
```

次にエンコードしたメッセージを送信する処理を実装します。
c_send_osc_message関数をmruby/cのsend_osc_messageとして定義します。
メッセージの送信にはesp-idfの[udp_clientのサンプル](https://github.com/espressif/esp-idf/tree/master/examples/protocols/sockets/udp_client)を参考にしました。

esp32/main/main.c

```c

static const char *TAG = "laser_harp";
static struct sockaddr_in dest_addr;
int sock;

(中略)

static void c_send_osc_message(mrb_vm *vm, mrb_value *v, int argc)
{
  unsigned char *payload = GET_STRING_ARG(1);
  size_t size = GET_INT_ARG(2);

  while (1)
  {
    int err = sendto(sock, payload, size, 0, (struct sockaddr *)&dest_addr, sizeof(dest_addr));
    if (err < 0)
    {
      ESP_LOGE(TAG, "Error occurred during sending: errno %d", errno);
    }
    else
    {
      ESP_LOGI(TAG, "Message sent");
      break;
    }
  }

void app_main(void)
{
  .....

  mrbc_define_method(0, mrbc_class_object, "send_osc_message", c_send_osc_message);

  .....
}

}
```

esp32/mrblib/models/osc.rb

```ruby
class OSCClient
  def send(message)
    send_osc_message(message, message.size)
  end
end
```

### 4. ここまでの処理をまとめる

レーザーを表すレーザークラスを実装します。

esp32/mrblib/models/laser.rb

```ruby
class Laser
  def initialize(gpio_pin, adc_pin, name)
    @pt = Phototransistor.new(gpio_pin, adc_pin)
    @name = name
  end

  # レーザーを遮られた時にtrueを返す。
  def play?
    # レーザーが遮られた時は、3000より小さい値
    value = @pt.value
    puts "#{@name}: #{value}"
    value < 3000
  end
end

```

作成した処理を実行するループを作成します。

esp32/mrblib/loops/master.rb

```ruby
Wifi.setup
laser_1 = Laser.new(32, 34, 'laser_1')
laser_2 = Laser.new(33, 35, 'laser_2')
osc_client = OSCClient.new

loop do
  if laser_1.play?
    osc_client.send("/trigger/laser_1\x00\x00\x00\x00,\x00\x00\x00")
  end

  if laser_2.play?
    osc_client.send("/trigger/laser_2\x00\x00\x00\x00,\x00\x00\x00")
  end

  sleep 0.001
end
```

ESP32のプログラムは以上です。

ここまで出来たら、レーザーの位置を調整して、レーザーの光をフォトトランジスタに当てます。
以下のコマンドでESP32にプログラムを書き込みます。

<pre>
$ cd esp32
$ make flash monitor
</pre>

この状態でレーザーを遮ってみると、以下のような結果になるはずです。
<pre>
.....
laser_1: 3074
laser_2: 3082
laser_1: 142　　# ここでレーザー1を遮りました
I (153771) laser_harp: Message sent  #メッセージ送信
laser_2: 3079
laser_1: 3073
.....
</pre>

# Sonic Pi側（PC）の実装

送信されたメッセージを受け取って、音を鳴らす処理を実装します。

### Sonic Piのインストール

[公式サイト](https://sonic-pi.net/)からダウンロードしてインストールします。

### Sonic Piでのメッセージ受信設定

Sonic Piでリモート(ESP32)からのメッセージを受信には、
「Prefs > 入出力 > Networked OSC」の「OSCサーバーを有効にする」と「OSCメッセージをリモートから受信する」をチェックする必要があります。

![e86bb4da-feeb-7944-0c90-b32c0c807e23](https://user-images.githubusercontent.com/2444124/208229109-23cbb12f-3644-49e0-8aac-6b0e9a789930.png)

### メッセージを受信

Sonic Piで演奏するRubyプログラムを作成します。
以下のように記述することで、OSCのメッセージを受け取ることができます。

music.rb

```ruby
# レーザー1を遮られた時に送信されるメッセージを受信して音を鳴らす。
live_loop :laser_1 do
  use_real_time
  sync "/osc/trigger/laser_1"
  play 60 # Cの音を鳴らす
end

# レーザー2を遮られた時に送信されるメッセージを受信して音を鳴らす。
live_loop :laser_2 do
  use_real_time
  sync "/osc/trigger/laser_2"
  play 64 # Eの音を鳴らす
end

```

music.rbをsonic_piで実行します。

<pre>
# sonic_piを起動した上で以下のコマンドを実行。
$ cat music.rb | sonic_pi
</pre>

この状態でレーザーを遮ると、Sonic Piから音が鳴ることを確認できます。😃♪








