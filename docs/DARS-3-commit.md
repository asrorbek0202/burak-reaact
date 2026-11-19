
## 1-QADAM: Redux'da javon ochamiz

**Commit:** `e0ae430` feat: develop ProductsPage slice and selector

**Maqsad:** o'quvchi "slice nima, selector nima" ni o'z so'zi bilan aytib bera olsin.

### Doskada tushuntiring (5 daq)

> "Products sahifasi uchta narsani eslab qolishi kerak: **taomlar ro'yxati**, **tanlangan taom**, **restoran ma'lumoti**. Bularni saqlash uchun omborda yangi javon ochamiz."

`ProductsPageState` ni chizing:

```
productsPage (javon)
 ├─ products:      []       ← taomlar ro'yxati
 ├─ chosenProduct: null     ← tanlangan taom
 └─ restaurant:    null     ← restoran
```

### Birgalikda yozamiz (15 daq)

**1) Shakl** `lib/types/screen.ts`: "javonda nima bo'lishi kerak":
```ts
export interface ProductsPageState {
  restaurant: Member | null;
  chosenProduct: Product | null;
  products: Product[];
}
```

**2) Slice** `productsPage/slice.ts`: "javonga qo'yish usullari":
```ts
const initialState = { restaurant: null, chosenProduct: null, products: [] };

reducers: {
  setProducts: (state, action) => { state.products = action.payload; },
  // setChosenProduct, setRestaurant ham shunday
}
```

**3) Store** `store.ts`: "javonni omborga qo'shamiz":
```ts
reducer: { homePage: HomePageReducer, productsPage: ProductsPageReducer }
```

**4) Selector** `productsPage/selector.ts`: "javondan olish usuli":
```ts
export const retrieveProducts = createSelector(selectProductsPage, (p) => p.products);
```

### Tekshiramiz (3 daq)

Brauzer Console'ida (`redux-logger` yoqilgan) `productsPage` hozircha bo'sh ekanini ko'rsating:
`{ restaurant: null, chosenProduct: null, products: [] }`

### O'quvchilarga savollar

1. `initialState` da nima uchun `products: []` (bo'sh massiv), `null` emas?
   *Javob:* `.map()` bo'sh massivda xato bermaydi, `null` da xato beradi.
2. Slice va selector farqi nima?
   *Javob:* slice **yozadi**, selector **o'qiydi**.

### O'qituvchi uchun eslatma (xatodan o'rganish)

Bu commit'da `ChosenProduct.tsx` da `setChosenProduct: (data: Product[])` deb yozilgan. Aslida bitta taom keladi, ya'ni `Product` bo'lishi kerak. Keyingi commit'da (`78d08b8`) tuzatilgan. O'quvchilarga ayting: *"Xato qilish normal, muhimi uni topib tuzatish."*

---

## 2-QADAM: Soxta ro'yxatni haqiqiy ma'lumot bilan almashtiramiz

**Commit:** `a75cb36` fix: retrieve data and modify products component

**Maqsad:** o'quvchi `useEffect` + `Service` + `dispatch` zanjirini o'zi yozib chiqsin.

### Doskada tushuntiring (5 daq)

Eski kod:
```tsx
const products = [
  { productName: "Cutlet", imagePath: "/img/cutlet.webp" },   // ← qo'lda yozilgan
  ...
];
```
Yangi kod ma'lumotni **backend'dan** oladi. Savol bering: *"Sahifa ochilganda ma'lumotni qachon so'raymiz?"* → *"Sahifa ochilgan zahoti, bir marta"* → `useEffect(..., [])`.

### Birgalikda yozamiz (15 daq), 4 ta kichik qadam

**1) Soxta ro'yxatni o'chiramiz.**

**2) Ombordan o'qish** (kim ko'rsatadi?):
```tsx
const { products } = useSelector(productsRetriever);
```

**3) Omborga yozish** (kim saqlaydi?):
```tsx
const { setProducts } = actionDispatch(useDispatch());
```

**4) Sahifa ochilganda ma'lumot so'raymiz:**
```tsx
useEffect(() => {
  const product = new ProductService();
  product
    .getProducts({ page: 1, limit: 8, order: "createdAt",
                   productCollection: ProductCollection.DISH, search: "" })
    .then((data) => setProducts(data))        // kelgan ma'lumotni omborga
    .catch((err) => console.log(err));        // xato bo'lsa Console'ga
}, []);
```

**5) Kartani haqiqiy ma'lumot bilan to'ldiramiz:**

| Eski (qo'lda) | Yangi (haqiqiy) |
|---|---|
| `product.imagePath` | `` `${serverApi}/${product.productImages[0]}` `` |
| `badgeContent={20}` | `badgeContent={product.productViews}` |
| `{12}` | `{product.productPrice}` |
| `"Normal size"` | `sizeVolume` (ichimlik bo'lsa litr, aks holda size) |
| `key={index}` | `key={product._id}` |

### Tekshiramiz (3 daq)

- Brauzerda `/products` ni oching. **Haqiqiy taomlar** chiqdimi?
- Console'da `redux-logger`: `setProducts` action'ini toping. `next state` da `products` to'ldiganini ko'rsating.

### O'quvchilarga savollar

1. Nima uchun `key={product._id}` yaxshiroq, `key={index}` emas?
   *Javob:* ro'yxat o'zgarsa (filtr, saralash) indeks o'zgaradi, `_id` esa har doim o'sha taomniki.
2. `useEffect` da `[]` olib tashlansa nima bo'ladi?
   *Javob:* har chizilishda so'rov ketadi, cheksiz aylanishga olib kelishi mumkin. Buni **ko'rsatmang**, faqat og'zaki ayting.
3. Backend o'chiq bo'lsa nima ko'rinadi? *("Products are not available!")*

### O'qituvchi uchun eslatma

Shu commit'dagi `"litre"` va `"size"` orasida bo'sh joy yo'q (`"1litre"`, `"LARGEsize"`). Keyingi commit'da tuzatilgan (`" L"`, `" size"`). O'quvchilardan so'rang: *"Ekranda nima noto'g'ri ko'rinyapti?"*. Shunda ular o'zlari topishadi.

---

## 3-QADAM: Bitta taom sahifasi + qidiruv/filtr

**Commit:** `78d08b8` feat: modify ChosenProduct component logic

Bu commit eng katta, shuning uchun uni **ikki bo'lakka** bo'ling.

### 3A. Taomni bosganda uning sahifasi ochilsin (15 daq)

**Doskada:** URL ni chizing:
```
/products            → hamma taomlar
/products/65ab12...  → bitta taom   (65ab12... = taomning _id si)
```

**Yozamiz (qadam-baqadam):**

1. `Products.tsx` da karta bosilganda URL ga o'tamiz:
   ```tsx
   const history = useHistory();
   const chooseDishHandler = (id: string) => { history.push(`/products/${id}`); };
   // kartada:  onClick={() => chooseDishHandler(product._id)}
   ```
2. `ChosenProduct.tsx` da URL dan `id` ni olamiz:
   ```tsx
   const { productId } = useParams<{ productId: string }>();
   ```
3. Shu `id` bilan taomni so'raymiz (1-2 qadamdagi naqshning o'zi):
   ```tsx
   useEffect(() => {
     new ProductService().getProduct(productId).then((data) => setChosenProduct(data));
     new MemberService().getRestaurant().then((data) => setRestaurant(data));
   }, []);
   ```
4. Ma'lumot kelguncha bo'sh turamiz:
   ```tsx
   if (!chosenProduct) return null;
   ```
5. Soxta qiymatlarni almashtiramiz: `"Kebab"` → `{chosenProduct?.productName}`, `$12` → `{chosenProduct?.productPrice}`, rasm slayderi → `chosenProduct.productImages.map(...)`.

**Savol:** nima uchun `if (!chosenProduct) return null;` kerak?
*Javob:* birinchi chizilishda ombor hali bo'sh (`null`). Shartsiz `chosenProduct.productName` xato beradi.

### 3B. Filtr, qidiruv va pagination (15 daq)

**Eng muhim g'oya (doskaga katta yozing):**

> **Tugma faqat "holat"ni o'zgartiradi. Ma'lumotni `useEffect` o'zi qayta so'raydi.**

Rasm: [`docs/img/04-products-loop.svg`](img/04-products-loop.svg)

```tsx
const [productSearch, setProductSearch] = useState<ProductInquiry>({
  page: 1, limit: 8, order: "createdAt",
  productCollection: ProductCollection.DISH, search: "",
});

useEffect(() => {
  new ProductService().getProducts(productSearch).then((data) => setProducts(data));
}, [productSearch]);      // ← productSearch o'zgarsa, qayta so'raydi
```

Keyin har bir tugma uchun bitta funksiya:

| Tugma | Nima o'zgaradi |
|---|---|
| Dish / Salad / Drink... | `productCollection` |
| New / Price / Views | `order` |
| Search | `search` |
| Pagination 1, 2, 3 | `page` |

Hammasi shu uchtasi bilan tugaydi: **holatni o'zgartir → `setProductSearch({ ...productSearch })`**.

**Savol:** nima uchun `{ ...productSearch }`, ya'ni nusxa yaratamiz?
*Javob:* React obyektning **yangi** bo'lganini ko'rsa, o'zgarish bo'ldi deb tushunadi.

> Eslatma: kodda hozir `productSearch.page = 1` deb eski obyekt to'g'ridan-to'g'ri o'zgartirilib, keyin nusxasi berilgan. Ishlaydi, lekin yangilarga **`setProductSearch({ ...productSearch, page: 1 })`** ni o'rgatgan yaxshiroq. Bu kodni o'zgartirish ixtiyoriy.

Yakunda `alert(collection)` ni ko'rsating. Bu tugma ishlayotganini sinash uchun yozilgan vaqtinchalik qator. Uni birgalikda o'chiring.

---

## Yakun (5 daq)

**Bugun o'rganganimizni 4 qatorda takrorlang** (o'quvchilar aytsin):

1. **Slice**: omborning javoni (boshlang'ich holat + yozish usullari)
2. **Selector**: ombordan o'qish
3. **Service + useEffect**: backend'dan ma'lumot olish (sahifa ochilganda)
4. **useState + useEffect[bog'liqlik]**: tugma bosilganda ma'lumotni qayta so'rash

**Tekshiruv savollari (qog'ozga yozdiring):**

1. `dispatch` nima qiladi? `useSelector` nima qiladi?
2. Sahifa ochilganda ma'lumotni qaysi hook bilan olamiz?
3. `useParams` nima uchun kerak?
4. Filtr tugmasi bosilganda ro'yxat qanday yangilanadi? (4 qadamni ayting)

## Uy vazifasi

| Daraja | Vazifa |
|---|---|
| Oson | Products sahifasida karta ostida taom **nomi** va **narxi** chiqishini tekshiring. `limit: 8` ni `limit: 4` qiling. |
| O'rta | `ChosenProduct` da `productDesc` bo'lmasa `"No Description"` chiqishini sinang. Qaysi qator shuni qiladi? |
| Qiyin | `productSearch.page = 1` uslubini `setProductSearch({ ...productSearch, page: 1, ... })` uslubiga o'tkazing. |

---

## Darsda tez-tez uchraydigan muammolar

| Muammo | Sabab | Yechim |
|---|---|---|
| Taomlar chiqmayapti, "not available" | Backend ishlamayapti yoki `.env` noto'g'ri | Backend'ni ishga tushiring, `.env` ni tekshiring, `npm start` ni qayta yoqing |
| Console'da `Network Error` | Backend manzili xato yoki CORS | `REACT_APP_API_URL` ni tekshiring |
| Rasmlar chiqmayapti | `serverApi` noto'g'ri | `${serverApi}/${...}` natijasini Console'ga chiqaring |
| Oq (bo'sh) ChosenProduct sahifasi | `return null` ishlayapti, ma'lumot kelmadi | Network tabida `/product/:id` so'rovini tekshiring |
| `.env` o'zgardi, lekin ta'sir qilmadi | React `.env` ni faqat ishga tushganda o'qiydi | `npm start` ni to'xtatib qayta yoqing |
