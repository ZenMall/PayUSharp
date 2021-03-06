Automatic LiveUpdate ödeme bilgilerinin işyeri tarafından kullanıcıdan alındığı ve işlenmek için sipariş bilgileri ile birlikte PayU'ya gönderildiği entegrasyon metodudur. Bu entegrasyonda kullanıcılar işyerinin sipariş sayfalarından hiç ayrılmazlar (3D Secure durumu hariç) ve ödeme Merchant ile PayU arasında bitirilir.

PayUSharp kütüphanesi ile bahsedilen sipariş ve ödeme bilgilerinin oluşturulması, gereken formatta PayU'ya iletilmesi ve PayU'dan dönen cevabın işlenmesi işlemleri kolaylaştırılmıştır.

### Başlangıç {#alustart}

Herhangi bir Automatic LiveUpdate işlemi gerçekleştirmeden önce yeni bir `PayU.AutomaticLiveUpdate.ALUService` nesnesi yaratılmalıdır. Bu nesneye geçirilmesi zorunlu olan tek parametre `signatureKey` alanıdir ve PayU'dan alınan imza anahtarı değeri geçirilmelidir.

Örnek kullanım şu şekildedir:

```cs
  var service = new PayU.AutomaticLiveUpdate.ALUService('signatureKey');
```

Ayrıca servisin testler sırasında kullanılabilecek `endpointUrl` ve `ignoreSSLCertificate` parametreleri de bulunmaktadır. Bu parametrelerin varsayılan değerlerini PayU tarafından tavsiye edilmedikçe değiştirmemeniz önerilir.

### Automatic LiveUpdate Siparişinin Oluşturulması

Automatic LiveUpdate sipariş bilgileri PayUSharp kütüphanesinde `PayU.AtomaticLiveUpdate.OrderDetails` sınıfı ile temsil edilmektedir. Yeni bir `PayU.AtomaticLiveUpdate.OrderDetails` nesnesi yaratılarak ve bu nesnenin [alanları](#alufields) sipariş bilgileri ile doldurularak bir PayUSharp Automatic LiveUpdate siparişi oluşturulur:

```cs
var order = new PayU.AutomaticLiveUpdate.OrderDetails();

order.Merchant = "PAYUDEMO";

order.OrderRef = "EXT_6112457";
order.OrderDate = DateTime.Now;

var product1 = new PayU.AutomaticLiveUpdate.ProductDetails
{
    Code = "TCK1",
    Name = "Ticket1",
    Quantity = 1,
    UnitPrice = 5.00M,
    Information = "Barcelona flight"
};

var product2 = new PayU.AutomaticLiveUpdate.ProductDetails
{
    Code = "TCK2",
    Name = "Ticket2",
    Quantity = 1,
    UnitPrice = 10.00M,
    Information = "London Flight"
};

order.ProductDetails.Add(product1);
order.ProductDetails.Add(product2);

order.PricesCurrency = "TRY";

order.CardDetails = new PayU.AutomaticLiveUpdate.CardDetails
{
    CardNumber = "4242424242424242",
    ExpiryMonth = "12",
    ExpiryYear = "2015",
    CVV = "000",
    CardOwnerName = "Ahmet Yılmaz"
};

order.BillingDetails = new PayU.AutomaticLiveUpdate.BillingDetails
{
    FirstName = "Ahmet",
    LastName = "Yilmaz",
    Email = "ahmet.yilmaz@payu.com.tr",
    PhoneNumber = "2122223344",
    CountryCode = "TR",
    Address = "Billing address", //optional
    Address2 = "Billing address ", //optional
    ZipCode = "12345", //optional
    City = "Kagithane", //optional - Ilce/Semt
    State = "Istanbul", //optional - Sehir
    Fax = "1234567890" //optional
};
order.DeliveryDetails = new PayU.AutomaticLiveUpdate.DeliveryDetails
{
    LastName = "John", //optional
    FirstName = "Doe", //optional
    Email = "shopper@payu.ro", //optional
    PhoneNumber = "1234567890", //optional
    Company = "Company Name", //optional
    Address = "Delivery Address", //optional
    Address2 = "Delivery Address", //optional
    ZipCode = "12345", //optional
    City = "City", //optional
    State = "State / Dept.", //optional
    CountryCode = "TR" //optional
};
order.ReturnUrl = "http://example.com/AutomaticLiveUpdate/ThreeDS.aspx";
order.ClientIpAddress = Request.UserHostAddress;
```

### Automatic LiveUpdate Siparişinin Gönderilmesi

Automatic LiveUpdate sürecinin tamamlanabilmesi için oluşturulmuş olan `PayU.AutomaticLiveUpdate.OrderDetails` tipindeki sipariş bilgisi sunucu tarafında PayU Automatic LiveUpdate adresine POST edilmelidir.

PayUSharp kütüphanesi bu POST işlemi için [Başlangıç](#alustart) adımında yarattığımız `ALUService` nesnesini kullanmaktadır. Sipariş bilgisi bu nesnenin `ProcessPayment` metoduna verilerek bu POST işleminin gerçekleşmesi sağlanır:

```cs
  var response = service.ProcessPayment(order);
```

**DİKKAT:** Bu metod çağırıldığında sunucu kodu PayU Automatic LiveUpdate sayfası ile HTTP POST üzerinden iletişime geçer ve PayU tarafından işlem yapılıp cevap dönene kadar bu metod dönmez. Bu işlemin kullanıcıyı sayfada bekleteceği ve sunucu tarafında request block edeceği unutulmamalıdır.

### Automatic LiveUpdate Sipariş Sonucunun İşlenmesi

`ProcessPayment` metodunun çağırılması sonucunda dönen cevap `PayU.AutomaticLiveUpdate.ALUResponse` tipinden olacaktır. Bu cevap nesnesi gerçekleşen veya hata veren ödeme sonucunu veren [alanlara](#alufields) sahiptir.

Normal akış dışında `ProcessPayment` metodu çağırıldığında iki olaya dikkat etmek gerekmektedir:

1. PayU ile iletişim sırasında bir hata olması durumunda `PayU.PayuException` tipinden bir exception alınacaktır. Bu nedenle bu metod çağrısı sırasında `PayU.PayuException` hataları yakalanmalı ve uygun şekilde işlenmelidir.
2. Eğer ödeme işlemi 3D-Secure ile yapılması gerekiyorsa o zaman `PayU.AutomaticLiveUpdate.ALUResponse` nesnesinin `Is3DSResponse` alanı `true` olacaktır. Bu durumda son kullanıcı `ALUResponse` nesnesinin `Url3DS` alanında döndürülen adrese yönlendirilmelidir. Bu adres PayU tarafından yönetilmektedir. Kullanıcı bu adreste 3D Secure işlemini tamamladıktan sonra PayU tarafından siparişte verilen `PayU.AutomaticLiveUpdate.OrderDetails` nesnesinin `ReturnUrl` alanındaki adrese geri yönlendirilecektir. Bu yönlendirme Automatic LiveUpdate sipariş sonucunu barındırmakta ve bu sonuç `PayU.AutomaticLiveUpdate.ALUResponse.FromHttpRequest` metodu kullanarak alınmaktadır.

Tipik bir Automatic LiveUpdate request/response şu şekilde olmalıdır:

#### Ödeme Sayfası - http://example.com/AutomaticLiveUpdate/Default.aspx

```cs
  public partial class Default : System.Web.UI.Page
  {
    public void Page_Load() {
      var service = new PayU.AutomaticLiveUpdate.ALUService('signatureKey');

      var order = new PayU.AutomaticLiveUpdate.OrderDetails();

      // ...
      // order alanlari atanir
      // ...

      // 3D Secure zorunlu ise 3D Secure islemi sonrasi sonucun donmesini istedigimiz adres
      order.ReturnUrl = "http://example.com/AutomaticLiveUpdate/ThreeDS.aspx";

      try
      {
        var response = service.ProcessPayment(parameters);

        if (response.Is3DSResponse) {
          // 3D Secure zorunlu. Kullaniciyi verilen adrese yonlendirmeliyiz.
          Response.Redirect(response.Url3DS);
          Response.End();
        }

        // Buraya kadar geldiysek ALU'dan cevap gelmis demektir.
        // response nesnesinin alanlarina bakarak sonucu anlayabiliriz.
        var status = response.Status

        if (status == Status.Success)
        {
          // Basarili bir islem durumu
        }
      }
      catch (PayuException ex)
      {
        // Hata islemleri yapilir
      }
    }
  }
```

#### 3D Secure Sayfası - http://example.com/AutomaticLiveUpdate/ThreeDS.aspx

```cs
  public partial class ThreeDS : System.Web.UI.Page
  {
    public void Page_Load() {
      // 3D Secure sonucu cevap gelmis. Cevap alanlari POST edilmis olarak gelecegi
      // icin bu alanlari kullanarak yeni bir ALUResponse yaratmaliyiz.
      var response = ALUResponse.FromHttpRequest (Request);

      // Buraya kadar geldiysek ALU'dan cevap gelmis demektir.
      // response nesnesinin alanlarina bakarak sonucu anlayabiliriz.
      var status = response.Status

      if (status == Status.Success)
      {
        // Basarili bir islem durumu
      }
    }
  }
```

### Automatic LiveUpdate Alanları {#alufields}

<<[fields/alu_fields.md]
