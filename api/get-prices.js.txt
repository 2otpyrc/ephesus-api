// Bu kod, Vercel'in sunucularında çalışacak olan Node.js kodudur.

export default async function handler(request, response) {
  // --- CORS Ayarları (ÇOK ÖNEMLİ) ---
  // Bu başlıklar, sadece sizin sitenizden gelen isteklere izin verir.
  response.setHeader('Access-Control-Allow-Origin', 'https://ephesus.site');
  response.setHeader('Access-Control-Allow-Methods', 'POST, OPTIONS');
  response.setHeader('Access-Control-Allow-Headers', 'Content-Type');

  // Tarayıcılar, asıl POST isteğinden önce bir OPTIONS "ön kontrol" isteği gönderir.
  // Bu isteğe 200 OK cevabı vererek devam etmesini sağlıyoruz.
  if (request.method === 'OPTIONS') {
    return response.status(200).end();
  }

  // Sadece POST metoduyla gelen isteklere izin ver.
  if (request.method !== 'POST') {
    return response.status(405).json({ message: 'Method Not Allowed' });
  }

  // Gizli anahtarları GÜVENLİ ortam değişkenlerinden al.
  const BOKUN_ACCESS_KEY = process.env.BOKUN_ACCESS_KEY;
  const BOKUN_SECRET_KEY = process.env.BOKUN_SECRET_KEY;

  // Sitemizden gönderilen ürün ID'lerini isteğin gövdesinden (body) al.
  const { ids } = request.body;

  if (!ids || !Array.isArray(ids)) {
    return response.status(400).json({ message: 'Product IDs are required.' });
  }

  // Bokun.io'nun API adresi (YAML dosyasında bu endpoint yok, bu yüzden varsayılanı kullanıyoruz. Gerekirse değiştirin.)
  const BOKUN_API_URL = 'https://api.bokun.io/api/20240101/products/pricing/product-list';

  try {
    // Bokun.io API'sine ASIL isteği yap.
    const bokunResponse = await fetch(BOKUN_API_URL, {
      method: 'POST',
      headers: {
        'Accept': 'application/json',
        'Content-Type': 'application/json',
        // YAML'e göre güncellenmiş DOĞRU anahtar başlıkları
        'access-key': BOKUN_ACCESS_KEY,
        'secret-key': BOKUN_SECRET_KEY
      },
      body: JSON.stringify({
        currency: "EUR",
        productIds: ids
      })
    });

    // Bokun.io'dan gelen cevabı direkt olarak al.
    const data = await bokunResponse.json();

    // Bokun.io'dan gelen cevap başarılı değilse, hatayı sitemize de ilet.
    if (!bokunResponse.ok) {
        // Bokun'dan gelen hata mesajını logla ve sitemize gönder.
        console.error('Bokun API Error:', data);
        return response.status(bokunResponse.status).json(data);
    }

    // Gelen veriyi işle ve sadece ihtiyacımız olanları (ID ve Fiyat) ayıkla.
    // DİKKAT: Bu kısım Bokun.io'nun döndürdüğü gerçek veri yapısına göre ayarlanmalıdır.
    const prices = data.map(item => ({
      id: item.productId,
      price: item.minPrice.amount.toFixed(2) // Fiyatı formatla
    }));

    // Ayıklanmış fiyat verisini sitemize geri gönder.
    response.status(200).json(prices);

  } catch (error) {
    console.error('API proxy error:', error);
    response.status(500).json({ message: 'An internal server error occurred.' });
  }
}