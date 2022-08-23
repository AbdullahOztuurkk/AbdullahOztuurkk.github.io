---
title:  "Dependency Injection Nedir?"
cover: /img/dependency-injection-nedir/index.png
date:   2020-10-15
tags:
- C#
- SOLID
- IoC
---
Merhaba, size ilk yazımda programcılar açısından çok önemli olan DI prensibini anlatmak istiyorum.SOLID prensiplerinin son harfi olan Dependency Injection tekniğinin amacı sınıflar arasında bağımlılıkları olabildiğince azaltmaktır. Size bunu örnekler üzerinden anlatmak istiyorum.

Mesaj gönderici servislerimiz olsun. Öncelikle hepsi için bir interface yazalım.

<!-- more -->

```csharp
public interface ISender
{
    public void SendMessage();
}
```
Şimdi bu interface’i implemente eden diğer class’larımızı yazalım.

```csharp
public class MailSender : ISender
{
    public void SendMessage()
    {
        Console.WriteLine("Mail Mesajı gönderildi.");
    }
}

public class SmsSender : ISender
{
    public void SendMessage()
    {
        Console.WriteLine("Sms Mesajı gönderildi.");
    }
}
```
Şimdi Dependency Injection tekniğinin uygulanmadığı yanlış tanımlamamızı yazalım.

```csharp
    public class MessageManager
    {
        SmsSender sender;
        public MessageManager()
        {
            sender = new SmsSender();
        }

        public void SendMessage()
        {
            sender.SendMessage();
        }
    }
```
Gördüğünüz gibi instance oluşturduğumuz gibi SmsSender class’ına bağımlı hale geldik.Peki neden? Ekibinizle yada sizin büyük bir projenizin olduğunu varsayalım.Bu class onlarca yerde kullanılıyor ve içerisinde de birden fazla metod olduğunu düşünelim. ( Basit tutmak açısndan sadece sendMessage metodunu yazdım. ) İlerleyen zamanlarda SmsSender sınıfında oluşacak değişikler , bu sınıfı kullandığımız tüm sınıfları etkilemekte olup tekrar gözden geçirmemiz gerekmekte. Başka bir örnekle ekibiniz yada iş vereniniz artık SmsSender yerine MailSender sınıfını kullanmanızı istiyor. Bu durumda bu sınıfları kullandığımız her bir kod bloğunu düzenlememiz gerekiyor. Bu gibi durumlarda bir sınıfa bağlı kalmak size ilerleyen zamanlarda ciddi vakit kaybi yaşatabilir.

Dependency Injection tekniğini kullanılarak sınıfımızı baştan yazalım.

```csharp
    public class MessageManager
    {
        private ISender _sender;
        public MessageManager(ISender sender)
        {
            _sender = sender;
        }

        public void SendMessage()
        {
            _sender.SendMessage();
        }
    }
```
Gelin, main metodu altında bu sınıfımızı kullanalım.

```csharp
static void Main(string[] args)
{
    MessageManager manager = new MessageManager(new SmsSender());
    manager.SendMessage();
    MessageManager manager = new MessageManager(new MailSender());
    manager.SendMessage();

    Console.ReadLine();
}
```

![Konsol Çıktısı][console-output]

Bağımlılığımızı en aza indirgeyerek uygulamamızı yazdık ve gördüğünüz üzere başarılı bir şekilde çalıştı.

Dependency Injection kullanarak projelerinize daha rahat yeni bir sınıf dahil edebilirsiniz ve olası bir değişikliğe karşı büyük bir vakit kaybının önüne geçebilirsiniz.Ninject,Autofac gibi birçok DI kütüphanesi bulunmakta. İnternetten bunlarla ilgili araştırma yapabilirsiniz.

<!-- Links -->
[console-output]: /img/dependency-injection-nedir/console-output.png