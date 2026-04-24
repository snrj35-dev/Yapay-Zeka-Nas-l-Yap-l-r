# Modern TypeScript: Architectural Mastery ve Type System Engineering

## 📌 Doküman Kartı

| Alan | Değer |
|---|---|
| Rol | TypeScript mimarisi, type-system tasarımı ve migration referansı |
| Durum | Living specification (`v1.5`) |
| Son güncelleme | 2026-04-03 |
| Birincil okur | TypeScript geliştiricileri, mimarlar, platform ekipleri |
| Ana girdi | Kod tabanı, domain modeli, migration hedefleri |
| Ana çıktı | Type-safe mimari kararlar, pattern seti, dönüşüm planı |
| Bağımlı dokümanlar | [ai-decision-execution-engine.md](ai-decision-execution-engine.md), [failure-pattern-atlas.md](failure-pattern-atlas.md), [cross-layer-integration-contract.md](cross-layer-integration-contract.md), [rag-source-quality-rubric.md](rag-source-quality-rubric.md) |

**Kalite notu:** Örneklerin bir kısmı kavramsal anlatım amaçlıdır. Uygulama sırasında proje `tsconfig`, runtime validator ve test stratejisi ile birlikte ele alınmalıdır.

---

## 1. TypeScript 5.0+ ve Runtime Meta-Programming

TypeScript 5.0 ile Decorators (Stage 3) artık standart hale geldi. Bu, sadece bir sözdizimi değişikliği değil, runtime ve compile-time arasındaki köprünün yeniden kurulmasıdır. Kurumsal mimarilerde decorators, **Cross-Cutting Concerns** (Loglama, Caching, Yetkilendirme) dediğimiz yapıları ana iş mantığından ayırmak için kullanılır.

> **Kod Durumu:** `Reference`
```typescript
function LogMethod(target: any, context: ClassMethodDecoratorContext) {
    const methodName = String(context.name);
    return function (this: any, ...args: any[]) {
        console.log(`Metot çağrıldı: ${methodName}`);
        return target.apply(this, args);
    };
}

// Method bazlı Caching Decorator - AOP Pattern
function Cache(ttl: number) {
    return function (target: any, context: ClassMethodDecoratorContext) {
        const cache = new Map();
        return function (this: any, ...args: any[]) {
            const key = JSON.stringify(args);
            if (cache.has(key)) return cache.get(key);
            
            const result = target.apply(this, args);
            cache.set(key, result);
            setTimeout(() => cache.delete(key), ttl);
            return result;
        };
    };
}

class UserService {
    @LogMethod
    @Cache(5000) // 5 saniye cache
    getUser(id: string) {
        // Expensive database operation
        return database.findUserById(id);
    }
}
```

### Const Type Parameters

Fonksiyon seviyesinde as const yazma zorunluluğunu ortadan kaldırır. Bu, özellikle readonly konfigürasyonlar için kritik bir imtiyazdır.

> **Kod Durumu:** `Reference`
```typescript
function routes<const T extends string[]>(t: T) {
    return t;
}
const r = routes(["home", "about"]); // r: readonly ["home", "about"]
```

## 2. Satisfies Operatörü (TS 4.9+ ve 5.0'ın Gizli Kahramanı)

Makalede as const ve Branded Types var ama satisfies eksik. Bu operatör, bir objenin bir tipi karşıladığından emin olmanı sağlar ama objenin kendi spesifik tip bilgisini (literal types) kaybetmez. as kullanmadan tip güvenliği sağlamanın en modern yoludur.

> **Kod Durumu:** `Reference`
```typescript
interface ThemeConfig {
  primary: string;
  secondary: string;
  borderRadius: number;
}

// ❌ as ile tip bilgisi kaybolur
const config1 = {
  primary: "#ff0000",
  secondary: "#00ff00",
  borderRadius: 8,
  extra: "bu property kaybolur"
} as ThemeConfig;

// ✅ satisfies ile tip bilgisi korunur
const config2 = {
  primary: "#ff0000",
  secondary: "#00ff00",
  borderRadius: 8,
  extra: "bu property korunur"
} satisfies ThemeConfig;

// config2.primary // "#ff0000" literal tip olarak korunur
```

## 3. Runtime Validation (Zod/Valibot Köprüsü)

En büyük "Senior" hatası şudur: TypeScript tipinin runtime'da (çalışma zamanında) var olduğunu sanmak. Makalede "Branded Types" ile email doğrulaması yaptık ama bu sadece "type assertion" ile oluyor. Gerçek dünyada "Static Types vs Runtime Validation" farkını ve bu ikisini Zod gibi kütüphanelerle nasıl senkronize tutacağımızı (Single Source of Truth) işlemeliyiz.

> **Kod Durumu:** `Reference`
```typescript
import { z } from 'zod';

// 1. Schema tanımı (Single Source of Truth)
const UserSchema = z.object({
  id: z.string().uuid(),
  email: z.string().email(),
  age: z.number().min(18)
});

// 2. TypeScript tipi otomatik olarak çıkar
type User = z.infer<typeof UserSchema>;
// type User = {
//   id: string;
//   email: string;
//   age: number;
// }

// 3. Runtime validation + tip güvenliği
function validateUser(data: unknown): User {
  return UserSchema.parse(data);
}

// 4. API'den gelen veri güvenli bir şekilde parse edilir
const apiResponse = await fetch('/api/user');
const userData = await apiResponse.json();
const user = validateUser(userData); // Runtime'da validation, compile-time'da tip güvenliği
```

## 4. Interface vs Type (Performans Farkı)

Compiler Internals bölümünde değinebiliriz: TypeScript derleyicisi için interface çözmek, type çözmekten daha hızlıdır. Çünkü interfaceler "caching" mekanizmasını kullanabilirken, type aliaslar her seferinde yeniden hesaplanabilir. Büyük projelerde bu, derleme süresini dakikalardan saniyelere indirebilir.

| Özellik | Interface | Type Alias |
|---------|-----------|------------|
| Performans | Cached, daha hızlı | Her seferinde hesaplanır |
| Inheritance | `extends` destekler | `&` ile birleştirme |
| Declaration Merging | Evet | Hayır |
| Recursive | Evet | Sınırlı |
| Use Case | Object shapes, contracts | Unions, utilities, computed types |

> **Kod Durumu:** `Reference`
```typescript
// ✅ Interface - daha performanslı
interface User {
  id: string;
  name: string;
}

// ❌ Type alias - daha yavaş (büyük projelerde fark edilir)
type User = {
  id: string;
  name: string;
};
```

## 5. Type-Level Debugging Teknikleri

İleri seviye patternlar yazarken (örneğin o yazdığımız DeepFlatten), bazen neden any döndüğünü anlamak imkansızlaşır.

### Utility Debugging

Bir tipi bir değişkene atayıp üzerine gelmek (hover) yetmediğinde, Expect<Equal<A, B>> gibi test tipleriyle nasıl debug yapılır?

> **Kod Durumu:** `Reference`
```typescript
// Debug utility types
type Expect<T extends true> = T;
type Equal<X, Y> = (<T>() => T extends X ? 1 : 2) extends (<T>() => T extends Y ? 1 : 2) ? true : false;
type NotEqual<X, Y> = true extends Equal<X, Y> ? false : true;

// Debug örnekleri
type Test1 = Expect<Equal<string, string>>; // ✅ Derlenir
type Test2 = Expect<Equal<string, number>>; // ❌ Hata verir

// Complex type debugging
type DeepFlattenDebug<T> = T extends readonly (infer U)[]
  ? U extends readonly unknown[]
    ? DeepFlattenDebug<U>
    : U
  : T;

// Test etmek için:
type TestNested = [1, [2, [3, [4]]]];
type TestFlat = DeepFlattenDebug<TestNested>; // Sonucu görmek için üzerine gel
type TestResult = Expect<Equal<TestFlat, 1 | 2 | 3 | 4>>; // Beklediğimiz mi?
```

## 6. Template Literal Types ile String Manipülasyonu

Sadece ön ek eklemek (on${Click}) değil, bir string'in içindeki parçaları parse etmek (örneğin bir URL içindeki parametreleri tipe dönüştürmek) gerçek bir "Advanced" konusudur:

> **Kod Durumu:** `Reference`
```typescript
type ExtractRouteParams<T extends string> = 
  T extends `${string}/:${infer Param}/${infer Rest}` 
    ? Param | ExtractRouteParams<`/${Rest}`>
    : T extends `${string}/:${infer Param}` 
      ? Param
      : never;

// URL route parametrelerini otomatik çıkar
type UserRoute = "/api/users/:id/posts/:postId";
type UserParams = ExtractRouteParams<UserRoute>; // "id" | "postId"

// CSS class names generation
type BEM<Block extends string, Element extends string, Modifier extends string> = 
  `${Block}${Element extends '' ? '' : `__${Element}`}${Modifier extends '' ? '' : `--${Modifier}`}`;

type ButtonVariants = BEM<'button', 'icon', 'primary'>; // "button__icon--primary"

// API endpoint validation
type ApiEndpoint = `/api/${string}/${number}`;
type ValidEndpoint = "/api/users/123"; // ✅ Geçerli
type InvalidEndpoint = "/api/users/abc"; // ❌ Derleme hatası
```

## 7. Teknik Derinlik: Variadic Tuples ve Recursive Flattening

Tip seviyesinde manipülasyon yaparken dizilerin derinliklerine inmek performans maliyeti getirir, ancak API tutarlılığı için paha biçilemezdir.

### DeepFlatten ve Chaining

Dizileri düzleştiren (flatten) özyinelemeli tipler, özellikle legacy API'lerden gelen karmaşık verileri normalize ederken kullanılır.

> **Kod Durumu:** `Reference`
```typescript
type DeepFlatten<T> = T extends readonly (infer U)[]
  ? DeepFlatten<U>
  : T;

type Nested = [1, [2, [3, [4]]]];
type Flat = DeepFlatten<Nested>; // 1 | 2 | 3 | 4
```

### Variadic Tuple Types

Dizileri birleştirirken (concat) veya kopyalarken kesin uzunluk ve içerik bilgisini korumamızı sağlar.

> **Kod Durumu:** `Reference`
```typescript
type Concat<T extends readonly unknown[], U extends readonly unknown[]> = [...T, ...U];
type Combined = Concat<[number, string], [boolean]>; // [number, string, boolean]
```

## 3. Karar Ağacı: Ne Zaman, Hangisini Kullanmalı?

En büyük hata, her soruna aynı tip çözümüyle yaklaşmaktır. İşte mühendislik perspektifinden karar ağacı:

### Branded Types vs. Discriminated Unions

| Özellik | Branded Types (Nominal) | Discriminated Unions (Structural) |
|---------|------------------------|-----------------------------------|
| Kullanım Amacı | İlkel tiplerin (string, number) karışmasını engellemek. | Farklı mantıksal dalları/durumları yönetmek. |
| Örnek | UserId vs ProductId (ikisi de string olsa bile). | SuccessResponse vs ErrorResponse. |
| Trade-off | Sürekli as Brand casting gerektirir, zahmetlidir. | Tip genişledikçe switch blokları büyür. |
| Karar | Veri tutarlılığı kritikse (örn: Finansal ID'ler) kullan. | İş akışı ve State yönetiliyorsa kullan. |

## 4. Performance ve Compiler Internals

TypeScript derleyicisi (TSC), karmaşık tiplerde O(n^n) karmaşıklığına çıkabilir.

### Circular Reference Handling

Çok derin DeepPartial veya recursive tipler Type instantiation is excessively deep hatasına yol açar. Çözüm: Tip derinliğini sınırlayın veya interface kullanarak "lazy evaluation" (tembel değerlendirme) yapın.

### Compilation Time Optimization

- any kullanımını minimize edin (Compiler her şeyi baştan kontrol eder).
- Karmaşık tipleri küçük type alias'lara bölün.
- skipLibCheck: true ile nodeModules içindeki kütüphaneleri taramayı durdurun.

## 5. Mimari Pattern'lar: DDD ve Event Sourcing

### Domain-Driven Design (DDD) ile Tip Güvenliği

Domain modellerinde "Value Objects" oluştururken Branded Types kullanarak Email ve Password stringlerinin yanlışlıkla birbirine atanmasını önleriz.

### Microservice Type Contracts

Mikroservisler arası iletişimde Shared Type Contracts paketi oluşturmak, API değişikliklerinde (breaking changes) diğer servislerin derleme aşamasında hata almasını sağlayarak sistemin çökmesini engeller.

> **Kod Durumu:** `Reference`
```typescript
// Shared Package
export type UserContract = {
  version: "1.0.0";
  data: { id: string; email: Email };
}
```

## 6. Real-World Trade-offs: Gerçekçi Bir Yaklaşım

### Mükemmel Tip vs. Geliştirme Hızı

Her şeyi %100 tip güvenli yapmak (Type-level programming), bazen ekibin kod yazma hızını %50 düşürebilir.

- Kural: Eğer yazdığın tip, başka bir geliştiricinin 30 saniyeden fazla hata mesajını anlamaya çalışmasına neden oluyorsa, o tipi basitleştir.
- Hata Mesajı Özelleştirme: Error tipleri oluşturarak geliştiricilere This action is only allowed for Admin types gibi özel uyarılar çıkarın.

## 7. Variance (Varyans) Derinlemesine: Tip Hiyerarşisinde Akış

Varyans, karmaşık tipler arasındaki alt tip (subtype) ilişkisinin nasıl değiştiğini açıklar. TypeScript'te bu konu genellikle fonksiyon parametreleri ve dönüş tiplerinde karşımıza çıkar.

### Covariance (Eş-değişkenlik): Çıktı Güvenliği

Bir tipin, daha spesifik bir tip (subtype) ile yer değiştirebilmesidir. TypeScript'te fonksiyon dönüş tipleri covariant'tır.

> **Kod Durumu:** `Reference`
```typescript
interface Animal { name: string }
interface Dog extends Animal { breed: string }

type Getter<T> = () => T;

let dogGetter: Getter<Dog> = () => ({ name: "Rex", breed: "German Shepherd" });
let animalGetter: Getter<Animal> = dogGetter; // ✅ OK: Dog, Animal'ın bir alt kümesidir.
```

### Contravariance (Zıt-değişkenlik): Girdi Güvenliği

Parametrelerde durum tersine döner. Bir fonksiyonun parametresi daha "geniş" bir tipi kabul etmelidir ki güvenli olsun.

> **Kod Durumu:** `Reference`
```typescript
type Logger<T> = (arg: T) => void;

let animalLogger: Logger<Animal> = (a) => console.log(a.name);
let dogLogger: Logger<Dog> = animalLogger; // ✅ OK: Her Animal'ı loglayabilen, her Dog'u da loglayabilir.

// ❌ Ancak tersi güvenli değildir:
// let catLogger: Logger<Animal> = (c: Cat) => c.meow(); // Hata! Çünkü her animal kedi değildir.
```

## 8. Recursive Types: Sınırsız Derinlikte Tip Tanımlama

Gerçek dünyada veriler (özellikle JSON) ağaç yapısındadır. TypeScript'te bir tipi kendi içinde çağırarak bu yapıları modelleyebiliriz.

### JSON Parsing Tipleri

Bir API'den gelebilecek her türlü geçerli JSON yapısını temsil eden tip:

> **Kod Durumu:** `Reference`
```typescript
type JSONValue = 
  | string 
  | number 
  | boolean 
  | null 
  | { [key: string]: JSONValue } 
  | JSONValue[];

const apiResult: JSONValue = {
  id: 1,
  metadata: {
    tags: ["typescript", "advanced"],
    active: true
  }
};
```

### DeepPartial (Derin Opsiyonel Yapı)

Partial<T> sadece ilk seviyeyi opsiyonel yapar. Tüm ağacı opsiyonel yapmak için özyineleme şarttır:

> **Kod Durumu:** `Reference`
```typescript
type DeepPartial<T> = {
  [P in keyof T]?: T[P] extends (infer U)[]
    ? DeepPartial<U>[]
    : T[P] extends object
    ? DeepPartial<T[P]>
    : T[P];
};

interface Config {
  db: { host: string; port: number };
  features: string[];
}

const partialUpdate: DeepPartial<Config> = {
  db: { host: "localhost" } // Port zorunlu değil!
};
```

## 9. Real-World Case Study: Tip Güvenli Event Bus

Bu tekniklerin hepsini birleştiren gerçek bir senaryo düşünelim: Bir uygulamanın içindeki olayları (events) yöneten, tamamen tip güvenli bir sistem.

### Senaryo: Veri Odaklı Olay Yönetimi

> **Kod Durumu:** `Reference`
```typescript
// 1. Olay Sözleşmesi (Contract)
interface AppEvents {
  "user:login": { id: string; name: string };
  "user:logout": { reason: string };
  "system:error": { code: number; message: string };
}

// 2. Generic Event Bus Sınıfı
class TypedEventBus<T> {
  private listeners: { [K in keyof T]?: ((data: T[K]) => void)[] } = {};

  // Contravariance kullanımı: Listener veriyi tüketir (input)
  on<K extends keyof T>(event: K, callback: (data: T[K]) => void): void {
    if (!this.listeners[event]) this.listeners[event] = [];
    this.listeners[event]?.push(callback);
  }

  emit<K extends keyof T>(event: K, data: T[K]): void {
    this.listeners[event]?.forEach(cb => cb(data));
  }
}

// 3. Uygulama
const bus = new TypedEventBus<AppEvents>();

bus.on("user:login", (user) => {
  console.log(`Hoş geldin ${user.name}`); // 'user' otomatik olarak {id, name} tipindedir.
});

// bus.emit("user:login", { id: "1" }); // ❌ Hata: 'name' eksik!
```

## 12. Makaleye Eklenmesi Gereken Karar Matrisi (Trade-off Matrix)

Bence en büyük eksiklik, okuyucuya şu kararı verdirecek net bir tablo:

| Senaryo | Önerilen Yöntem | Neden? |
|---------|----------------|--------|
| Karmaşık API Yanıtları | Zod + infer | Runtime güvenliği ve tip senkronizasyonu için. |
| Finansal/Kritik ID'ler | Branded Types | productId ile orderId karışırsa sistem çöker. |
| UI State Yönetimi | Discriminated Unions | switch-case ile tüm caseleri kapsama garantisi. |
| Kütüphane/SDK Yazımı | Generics + infer | Kullanıcıya esneklik ve otomatik tamamlama sunmak için. |
| Büyük Projeler | Interface over Type | Derleme performansı ve declaration merging için. |
| Configuration Objects | satisfies + const | Tip güvenliği + literal type preservation. |

## 14. Expert Level: Type System Engineering

### 14.1. Type-Level State Machines

State management'i tip seviyesinde modellemek, runtime hatalarını tamamen ortadan kaldırır:

> **Kod Durumu:** `Reference`
```typescript
type State = 'idle' | 'loading' | 'success' | 'error';
type Event = 
  | { type: 'FETCH' }
  | { type: 'SUCCESS'; data: unknown }
  | { type: 'ERROR'; error: string }
  | { type: 'RESET' };

type Transition<S, E> = 
  S extends 'idle' ? E extends { type: 'FETCH' } ? 'loading' : never :
  S extends 'loading' ? E extends { type: 'SUCCESS' } ? 'success' :
                       E extends { type: 'ERROR' } ? 'error' : never :
  S extends 'success' | 'error' ? E extends { type: 'RESET' } ? 'idle' : never :
  never;

type NextState<S extends State, E extends Event> = Transition<S, E>;

// Type-safe state machine
class StateMachine<S extends State> {
  private state: S;
  
  constructor(initialState: S) {
    this.state = initialState;
  }
  
  transition<E extends Event>(event: E): NextState<S, E> {
    // Runtime logic buraya gelir
    throw new Error('Not implemented');
  }
}
```

### 14.2. Type-Driven Development (TDD)

Test-driven development'in tip seviyesindeki karşılığı:

> **Kod Durumu:** `Reference`
```typescript
// 1. Önce tip'i tanımla
type ApiResponse<T> = {
  data: T;
  status: 200 | 400 | 404 | 500;
  message?: string;
};

// 2. Tip'i test et
type TestResponse = ApiResponse<{ id: string; name: string }>;
type TestValid = TestResponse['status']; // 200 | 400 | 404 | 500

// 3. Implementation
function createApiResponse<T>(
  data: T, 
  status: TestResponse['status'], 
  message?: string
): ApiResponse<T> {
  return { data, status, message };
}
```

### 14.3. Advanced Type Composition

Birden fazla tipi birleştirmek için composition pattern'leri:

> **Kod Durumu:** `Reference`
```typescript
// Higher-order types
type WithId<T> = T & { id: string };
type WithTimestamps<T> = T & { 
  createdAt: Date; 
  updatedAt: Date;
};

type BaseEntity<T> = WithId<WithTimestamps<T>>;

// Composable entity types
type User = BaseEntity<{
  name: string;
  email: string;
}>;

type Product = BaseEntity<{
  title: string;
  price: number;
}>;

// Type-level validation
type ValidateEntity<T> = 
  T extends { id: infer Id }
    ? Id extends string 
      ? T 
      : never
    : never;
```

### 14.4. Type-Level Algorithms

Tip seviyesinde algoritma yazmak:

> **Kod Durumu:** `Reference`
```typescript
// Fibonacci tip seviyesinde
type Fibonacci<N extends number> = 
  N extends 0 ? 0 :
  N extends 1 ? 1 :
  Add<Fibonacci<Sub<N, 1>>, Fibonacci<Sub<N, 2>>>;

// Type-level arithmetic (TypeScript 4.8+)
type Add<A extends number, B extends number> = 
  [...Tuple<A>, ...Tuple<B>]['length'] extends number ? 
  [...Tuple<A>, ...Tuple<B>]['length'] : never;

type Tuple<N extends number, T extends unknown[] = []> = 
  T['length'] extends N ? T : Tuple<N, [...T, unknown]>;

// Type-level string operations
type Split<S extends string, D extends string> = 
  S extends `${infer T}${D}${infer U}` 
    ? [T, ...Split<U, D>] 
    : [S];

type Join<T extends string[], D extends string> = 
  T extends [infer F extends string, ...infer R extends string[]]
    ? R extends []
      ? F
      : `${F}${D}${Join<R, D>}`
    : '';
```

### 14.5. Type System Performance Engineering

Büyük projelerde tip sistemi optimizasyonu:

> **Kod Durumu:** `Reference`
```typescript
// Lazy type evaluation
interface Lazy<T> {
  readonly _type: T;
}

// Type memoization patterns
type Memo<T> = T & { __memo: true };

// Recursive depth limiting
type DepthLimited<T, Max extends number = 10, Current extends number = 0> = 
  Current extends Max 
    ? 'DEPTH_LIMIT_EXCEEDED'
    : T extends object
      ? { [K in keyof T]: DepthLimited<T[K], Max, Add<Current, 1>> }
      : T;

// Performance benchmarks
interface TypeMetrics {
  compilationTime: number;
  memoryUsage: number;
  complexityScore: number;
}
```

## 15. Enterprise Architecture Patterns

### 15.1. Microservice Type Contracts

Distributed sistemlerde tip güvenliği:

> **Kod Durumu:** `Reference`
```typescript
// Service contract definitions
interface ServiceContract<T, R> {
  request: T;
  response: R;
  version: string;
}

// Typed RPC
type RpcCall<T extends ServiceContract<any, any>> = 
  (request: T['request']) => Promise<T['response']>;

// Service registry
interface ServiceRegistry {
  userService: ServiceContract<{ id: string }, User>;
  productService: ServiceContract<{ category: string }, Product[]>;
  orderService: ServiceContract<{ userId: string; productId: string }, Order>;
}

// Type-safe service client
class TypedClient<R extends ServiceRegistry> {
  async call<K extends keyof R>(
    service: K, 
    request: R[K]['request']
  ): Promise<R[K]['response']> {
    // Implementation
    throw new Error('Not implemented');
  }
}
```

### 15.2. Domain-Driven Type Design

Domain logic'i tip sistemi içinde modelleme:

> **Kod Durumu:** `Reference`
```typescript
// Value objects with invariant enforcement
abstract class ValueObject<T> {
  protected readonly value: T;
  
  constructor(value: T) {
    this.validate(value);
    this.value = value;
  }
  
  protected abstract validate(value: T): void;
}

// Domain-specific value objects
class Email extends ValueObject<string> {
  protected validate(value: string): void {
    if (!value.includes('@')) throw new Error('Invalid email');
  }
}

class Money extends ValueObject<{ amount: number; currency: string }> {
  protected validate(value: { amount: number; currency: string }): void {
    if (value.amount < 0) throw new Error('Amount cannot be negative');
  }
}

// Domain events with type safety
interface DomainEvent<T> {
  type: string;
  aggregateId: string;
  data: T;
  timestamp: Date;
}

// Event sourcing types
type EventStream<T extends DomainEvent<any>> = readonly T[];
type AggregateState<S, E extends DomainEvent<any>> = 
  (state: S, event: E) => S;
```

### 15.3. Type-Safe Configuration Management

Configuration'ı tip sistemi içinde yönetme:

> **Kod Durumu:** `Production-Ready`
```typescript
// Hierarchical configuration with type safety
interface ConfigSchema {
  database: {
    host: string;
    port: number;
    ssl: boolean;
  };
  redis: {
    url: string;
    ttl: number;
  };
  features: {
    newUI: boolean;
    betaAPI: boolean;
  };
}

// Environment-specific configs
type EnvironmentConfig<T> = {
  [K in keyof T]: T[K] extends object 
    ? EnvironmentConfig<T[K]> 
    : T[K] | string; // Allow env var substitution
};

// Config validation with Zod integration
const ConfigSchema = z.object({
  database: z.object({
    host: z.string(),
    port: z.number(),
    ssl: z.boolean()
  }),
  redis: z.object({
    url: z.string().url(),
    ttl: z.number()
  }),
  features: z.object({
    newUI: z.boolean(),
    betaAPI: z.boolean()
  })
});

type AppConfig = z.infer<typeof ConfigSchema>;
```

## 16. Advanced Tooling ve Ecosystem Integration

### 16.1. Custom Type Checkers

TypeScript'e özel type checking kuralları ekleme:

> **Kod Durumu:** `Reference`
```typescript
// Custom type guards with complex logic
function isCompleteUser(user: unknown): user is Required<User> {
  return (
    typeof user === 'object' &&
    user !== null &&
    'id' in user &&
    'name' in user &&
    'email' in user &&
    typeof user.id === 'string' &&
    typeof user.name === 'string' &&
    typeof user.email === 'string'
  );
}

// Type-level assertions
function assertType<T>(value: unknown, validator: (v: unknown) => v is T): asserts value is T {
  if (!validator(value)) {
    throw new TypeError('Type assertion failed');
  }
}
```

### 16.2. IDE Integration Patterns

VS Code ve diğer IDE'ler için tip-aware extensions:

> **Kod Durumu:** `Reference`
```typescript
// Code completion providers
interface CompletionProvider<T> {
  provideCompletions(context: T): string[];
}

// Type-aware refactoring
interface Refactoring<T> {
  canApply(node: T): boolean;
  apply(node: T): T;
}

// Linting rules with type awareness
interface TypeAwareRule {
  check(node: unknown, typeChecker: TypeChecker): RuleResult[];
}
```

## 17. Modern AI IDE ve RAG Entegrasyonu

### 17.1. RAG: Yardımcı Hafıza, Çözüm Motoru Değil

Modern AI IDE'lerinde RAG (Retrieval-Augmented Generation) tek başına bir çözüm motoru değildir; daha ziyade **yardımcı hafıza katmanıdır**. Cursor ve Windsurf gibi platformlarda gördüğümüz mimari şu şekilde işler:

> **Kod Durumu:** `Reference`
```
Context Builder → Retrieval → Reasoning → Tooling → Validation
```

**RAG'in Rolü:**
- Codebase indexing ve semantic search için hafıza katmanı
- Dokümantasyon ve knowledge retrieval
- Örnek kod parçacıklarını bulma

**RAG'in Yetersiz Olduğu Alanlar:**
- "Bu fonksiyon hangi dosyaları etkiler?" → AST/LSP/symbol table gerekir
- "Bu import zinciri nereyi kırar?" → Dependency analysis gerekir
- "Bu type gerçekten ne bekliyor?" → Type checker gerekir

### 17.2. TypeScript Compiler API ile Context Builder

Gerçek dünyada AI IDE'leri TypeScript Compiler API kullanarak zengin context oluşturur:

> **Kod Durumu:** `Reference`
```typescript
import * as ts from 'typescript';

// Context Builder için gerçekçi API kullanımı
class TypeScriptContextBuilder {
  private program: ts.Program;
  private checker: ts.TypeChecker;

  constructor(configPath: string) {
    const config = ts.readConfigFile(configPath, ts.sys.readFile);
    const parsedConfig = ts.parseJsonConfigFileContent(
      config.config, ts.sys, path.dirname(configPath)
    );
    
    this.program = ts.createProgram(
      parsedConfig.fileNames, 
      parsedConfig.options
    );
    this.checker = this.program.getTypeChecker();
  }

  // Node bazlı type sorgulama (doğru yaklaşım)
  getTypeAtPosition(filePath: string, position: number) {
    const sourceFile = this.program.getSourceFile(filePath);
    if (!sourceFile) return null;

    const node = this.getNodeAtPosition(sourceFile, position);
    if (!node) return null;

    const type = this.checker.getTypeAtLocation(node);
    return {
      type: this.checker.typeToString(type),
      symbol: this.checker.getSymbolAtLocation(node),
      definition: this.getDefinitionInfo(node)
    };
  }

  // Symbol table ve dependency analysis
  buildSymbolTable(filePath: string): SymbolInfo[] {
    const sourceFile = this.program.getSourceFile(filePath);
    const symbols: SymbolInfo[] = [];
    
    ts.forEachChild(sourceFile, (node) => {
      if (ts.isVariableDeclaration(node) || ts.isFunctionDeclaration(node)) {
        const symbol = this.checker.getSymbolAtLocation(node.name);
        if (symbol) {
          symbols.push({
            name: symbol.getName(),
            type: this.checker.typeToString(
              this.checker.getTypeOfSymbolAtLocation(symbol, node)
            ),
            declarations: symbol.getDeclarations()?.map(d => d.getSourceFile().fileName),
            usages: this.findUsages(symbol)
          });
        }
      }
    });

    return symbols;
  }

  // Git context entegrasyonu
  getGitContext(filePath: string): GitDiff[] {
    // Git diff ve history analysis
    // Bu kısım git entegrasyonu ile gerçekleştirilir
    return [];
  }
}

// Context yapısı
interface CodeContext {
  imports: Import[];
  symbols: SymbolInfo[];
  currentScope: ScopeInfo;
  gitHistory: GitDiff[];
  ast: ts.Node;
  typeChecker: ts.TypeChecker;
}
```

### 17.3. Multi-Modal Retrieval Mimarisi

Modern AI IDE'leri tek bir retrieval yöntemine bağlı kalmaz:

> **Kod Durumu:** `Reference`
```typescript
interface MultiModalRetrieval {
  // 1. Codebase index (semantic + structural)
  codebaseSearch: {
    semantic: string[]; // Embedding-based search
    structural: SymbolMatch[]; // AST-based search
    dependency: DependencyGraph; // Import/export analysis
  };

  // 2. Local/pattern search
  localSearch: {
    regex: PatternMatch[]; // Pattern-based search
    ast: ASTPattern[]; // Structural patterns
    linter: LinterRule[]; // Code quality patterns
  };

  // 3. External knowledge
  externalKnowledge: {
    documentation: DocMatch[]; // MDN, TypeScript docs
    examples: CodeExample[]; // Stack Overflow, GitHub
    bestPractices: PatternMatch[]; // Community patterns
  };
}
```

### 17.4. Context Quality ve Validation Loop

AI IDE performansı için kritik formül:

> **Kod Durumu:** `Reference`
```
AI IDE Power = Model Quality × Context Quality × Tool Access × Validation Loop
```

**Context Quality Components:**
- **Relevance:** Doğru dosyaları ve sembolleri bulma
- **Freshness:** Güncel kod ve dependency bilgisi
- **Completeness:** Import zincirleri ve etkilenen dosyalar
- **Accuracy:** Type checker ve LSP doğruluğu

**Validation Loop Mekanizmaları:**
> **Kod Durumu:** `Reference`
```typescript
interface ValidationLoop {
  // Pre-edit validation
  beforeEdit: {
    typeCheck: TypeCheckResult;
    lintCheck: LinterResult;
    testCoverage: CoverageResult;
  };

  // Post-edit validation
  afterEdit: {
    compilation: CompilationResult;
    runtimeTest: TestResult;
    performance: PerfImpact;
  };

  // Feedback to model
  feedback: {
    successRate: number;
    errorPatterns: ErrorPattern[];
    userCorrections: CorrectionData[];
  };
}
```

### 17.5. Real-World Marka Eşlemeleri (Esnek Yaklaşım)

Marka eşlemelerini "örnek yaklaşım" olarak sunmak daha doğrudur:

> **Kod Durumu:** `Reference`
```typescript
// Modern AI IDE Yaklaşımları (Değişebilir)
interface AIIDEApproaches {
  // GitHub Copilot
  copilot: {
    models: ["OpenAI Codex", "GPT-4", "Custom fine-tunes"];
    context: "GitHub data + local context";
    strengths: ["Code completion", "Documentation generation"];
  };

  // Cursor
  cursor: {
    models: ["GPT-4", "Claude", "Local models"];
    context: "Full codebase indexing + semantic search";
    strengths: ["Codebase understanding", "Refactoring"];
  };

  // Windsurf
  windsurf: {
    models: ["Mix of proprietary + open source"];
    context: "RAG-based context engine + local indexing";
    strengths: ["Context pinning", "Multi-file editing"];
  };

  // Codeium
  codeium: {
    models: ["Mixtral", "Custom models"];
    context: "Codebase search + real-time indexing";
    strengths: ["Free tier", "Enterprise features"];
  };
}
```

### 17.7. Popüler RAG Pattern'ları ve Gerçek Dünya Uygulamaları

Modern AI IDE'lerinde kullanılan RAG pattern'ları ve gerçek dünyadaki karşılıkları:

#### 1. Code Completion + Context (Core Feature)
> **Kod Durumu:** `Reference`
```typescript
// VS Code extension'lar için gerçekçi yaklaşım
interface CompletionContext {
  filePath: string;
  cursorPosition: Position;
  surroundingCode: string;
  imports: Import[];
  symbols: SymbolInfo[];
}

function getCodeCompletion(context: CompletionContext): CompletionItem[] {
  // 1. Local codebase search (AST + symbol table)
  const localMatches = symbolIndex.search(context.surroundingCode);
  
  // 2. RAG'dan ilgili chunk'ları çek (docs + examples)
  const relevantChunks = rag.search({
    query: context.surroundingCode,
    filters: { language: "typescript", useCase: "completion" }
  });
  
  // 3. LLM ile birleştir
  return generateCompletions(localMatches, relevantChunks);
}
```

#### 2. Error Explanation + Fix Suggestions
> **Kod Durumu:** `Reference`
```typescript
// Hata mesajlarını anlama ve çözüm önerme
interface ErrorContext {
  errorMessage: string;
  code: string;
  language: string;
  stackTrace?: string;
  typeInfo?: TypeInfo;
}

function explainError(error: ErrorContext): ErrorExplanation {
  // 1. Type checker'dan detaylı bilgi al
  const typeError = typeChecker.analyzeError(error);
  
  // 2. RAG'dan benzer hataları bul
  const similarErrors = rag.search(`TypeScript ${error.errorMessage} examples`);
  
  // 3. LLM ile reasoning
  return llm.explain({
    error: typeError,
    examples: similarErrors,
    context: error.code
  });
}
```

#### 3. Refactoring Suggestions (Hybrid Approach)
> **Kod Durumu:** `Reference`
```typescript
// Kod dönüşüm önerileri
interface RefactorContext {
  code: string;
  selection: Range;
  targetPattern: string;
  typeInfo: TypeInfo;
  dependencies: DependencyInfo[];
}

function suggestRefactor(context: RefactorContext): RefactorSuggestion[] {
  // 1. AST pattern matching
  const astPatterns = astAnalyzer.findPatterns(context.code);
  
  // 2. RAG pattern search
  const ragPatterns = rag.search({
    query: context.targetPattern,
    filters: { category: "refactoring", language: "typescript" }
  });
  
  // 3. Type safety check
  const validPatterns = typeChecker.validateTransformations(
    astPatterns, 
    context.typeInfo
  );
  
  return combineAndRank(validPatterns, ragPatterns);
}
```

#### 4. Hybrid Search Strategy (Production Critical)
> **Kod Durumu:** `Reference`
```typescript
// Keyword + Semantic + Structural search kombinasyonu
function hybridSearch(query: string, context: CodeContext): SearchResult[] {
  // 1. Keyword/BM25 search (hızlı)
  const keywordResults = keywordIndex.search(query);
  
  // 2. Semantic embedding search (anlamsal)
  const semanticResults = vectorStore.search({
    query: embed(query),
    filters: getContextFilters(context)
  });
  
  // 3. Structural/AST search (yapısal)
  const structuralResults = astIndex.search({
    pattern: query,
    context: context
  });
  
  // 4. Multi-factor ranking
  return mergeAndRank({
    keyword: keywordResults,
    semantic: semanticResults,
    structural: structuralResults,
    weights: { keyword: 0.3, semantic: 0.5, structural: 0.2 }
  });
}
```

#### 5. Context-Aware Retrieval (Advanced)
> **Kod Durumu:** `Reference`
```typescript
// Kullanıcının kod context'ine göre akıllı arama
function contextAwareSearch(
  userCode: string, 
  cursor: Position,
  fullContext: CodeContext
): Chunk[] {
  // 1. Immediate context (cursor çevresi)
  const nearbyCode = extractNearbyCode(userCode, cursor, 500);
  
  // 2. Symbol context (import chain)
  const symbolContext = analyzeSymbolContext(fullContext.symbols);
  
  // 3. Type context (current type information)
  const typeContext = analyzeTypeContext(fullContext.typeChecker);
  
  // 4. Git context (recent changes)
  const gitContext = getRecentChanges(fullContext.gitHistory);
  
  return rag.search({
    query: inferIntent(nearbyCode, symbolContext),
    filters: {
      language: "typescript",
      difficulty: inferDifficulty(fullContext),
      useCase: inferUseCase(typeContext),
      recency: gitContext.timeframe
    },
    boost: {
      userPatterns: getUserPatterns(),
      projectPatterns: getProjectPatterns()
    }
  });
}
```

### 17.8. Production İpuçları ve En İyi Pratikler

#### Multi-Modal RAG Architecture
> **Kod Durumu:** `Reference`
```typescript
interface MultiModalRAG {
  // 1. Code-specific embeddings
  codeEmbeddings: {
    model: "code-specialized-embedding", // CodeBERT, GraphCodeBERT
    chunking: "function-level",
    preprocessing: "ast-aware"
  };
  
  // 2. Documentation embeddings
  docEmbeddings: {
    model: "general-text-embedding",
    chunking: "paragraph-level",
    preprocessing: "markdown-aware"
  };
  
  // 3. Example embeddings
  exampleEmbeddings: {
    model: "code-example-embedding",
    chunking: "snippet-level",
    preprocessing: "context-preserved"
  };
}
```

#### Real-time Learning Pipeline
> **Kod Durumu:** `Reference`
```typescript
interface LearningPipeline {
  // 1. User feedback collection
  collectFeedback: {
    acceptedSuggestions: CompletionItem[];
    rejectedSuggestions: CompletionItem[];
    userEdits: CodeEdit[];
    errorCorrections: ErrorFix[];
  };
  
  // 2. Pattern extraction
  extractPatterns: {
    frequentPatterns: Pattern[];
    antiPatterns: AntiPattern[];
    userPreferences: UserPreference[];
  };
  
  // 3. RAG update
  updateRAG: {
    addNewChunks: Chunk[];
    updateWeights: WeightUpdate[];
    personalizeRanking: PersonalizationFactor[];
  };
}
```

#### Type System Integration (Critical)
> **Kod Durumu:** `Reference`
```typescript
interface TypeAwareRAG {
  // 1. Type-based filtering
  typeFiltering: {
    requiredTypes: Type[];
    forbiddenTypes: Type[];
    typeConstraints: TypeConstraint[];
  };
  
  // 2. Symbol-based retrieval
  symbolRetrieval: {
    relatedSymbols: Symbol[];
    symbolUsages: Usage[];
    symbolDependencies: Dependency[];
  };
  
  // 3. Generic-aware search
  genericSearch: {
    genericPatterns: GenericPattern[];
    typeParameters: TypeParameter[];
    constraints: Constraint[];
  };
}
```

### 17.9. Gerçek Dünya "Secret Sauce" Pipeline

Modern AI IDE'lerinin gerçek çalışma mantığı:

> **Kod Durumu:** `Reference`
```typescript
interface ProductionAIPipeline {
  // Phase 1: Context Builder (AST + LSP + Git)
  contextBuilder: {
    openFiles: SourceFile[];
    symbolTable: SymbolTable;
    importGraph: DependencyGraph;
    gitHistory: GitDiff[];
    typeInfo: TypeInfo[];
  };
  
  // Phase 2: Multi-Modal Retrieval
  retrievalLayer: {
    codebaseSearch: CodeSearchResult[];
    ragSearch: RAGResult[];
    externalDocs: Documentation[];
    examples: CodeExample[];
  };
  
  // Phase 3: LLM Reasoning
  reasoningLayer: {
    promptEngineering: PromptTemplate[];
    contextInjection: ContextChunk[];
    chainOfThought: ThoughtProcess[];
  };
  
  // Phase 4: Tool Integration
  toolingLayer: {
    languageServer: LSPServer;
    typeChecker: TypeChecker;
    linter: LinterResult[];
    testRunner: TestResult[];
  };
  
  // Phase 5: Validation Loop
  validationLayer: {
    preEditValidation: ValidationResult[];
    postEditValidation: ValidationResult[];
    feedbackCollection: UserFeedback[];
    qualityMetrics: QualityMetric[];
  };
}
```

**Kritik Ayrım:** 
- **RAG = Yardımcı hafıza** (docs, examples, patterns)
- **LLM = Ana beyin** (reasoning, synthesis)
- **LSP/AST = Doğruluk katmanı** (type safety, structure)

### 17.10. Pratik Uygulama: AI-Aware TypeScript Development

Geliştiriciler için pratik tavsiyeler:

1. **AI-Friendly Code Writing:**
   ```typescript
   // ✅ AI için anlaşılır, iyi isimlendirilmiş kod
   interface UserProfile {
     id: UserId; // Branded type, AI anlayabilir
     email: EmailAddress; // Domain-specific type
     preferences: UserPreferences; // Clear separation
   }

   // ❌ AI için kafa karıştırıcı kod
   interface D {
     i: string; // Anlamsız kısaltma
     e: string;
     p: any; // Type safety yok
   }
   ```

2. **Context Optimization:**
   ```typescript
   // AI için daha iyi context sinyalleri
   /**
    * User authentication service
    * Handles JWT tokens and session management
    * @see {@link UserProfile} for user data structure
    */
   class AuthService {
     // Clear method names and documentation
     async authenticateUser(credentials: LoginCredentials): Promise<AuthResult>;
   }
   ```

3. **Type Safety for AI:**
   ```typescript
   // AI'in yanlış type inference yapmasını engelle
   const config = {
     apiUrl: "https://api.example.com",
     timeout: 5000,
     retries: 3
   } satisfies AppConfig; // satisfies ile literal types korunur
   ```

4. **RAG-Optimized Documentation:**
   ```typescript
   /**
    * Processes user authentication with JWT tokens
    * 
    * @example
    * ```typescript
    * const auth = new AuthService();
    * const result = await auth.authenticateUser({
    *   email: "user@example.com",
    *   password: "secure123"
    * });
    * ```
    * 
    * @related {@link UserProfile}, {@link LoginCredentials}
    * @category Authentication
    * @since v1.0.0
    */
   ```

### 17.11.Local AI Kurulum Önerileri


> **Kod Durumu:** `Production-Ready`
```bash
# 1. Vector database (FAISS / Chroma)
pip install faiss-cpu chromadb

# 2. TypeScript specific tools
npm install -g typescript @types/node
npm install typescript-lsp

# 3. Embedding models
pip install sentence-transformers
# veya CodeBERT için:
pip install transformers torch

# 4. Local LLM (Ollama)
curl -fsSL https://ollama.ai/install.sh | sh
ollama pull codellama:7b
ollama pull deepseek-coder:6.7b
```

**Eksik olan bileşenler:**
- AST parsing library (`@typescript-eslint/parser`)
- Symbol index builder
- Git integration
- Real-time embedding generation

---

## 18. TypeScript as AI Control Plane

### **🎯 TypeScript = AI Sistemin Beyni (Control Plane)**

---

## 🤖 **1. AI-Native Type System**

### **📘 AI Response Contract**
> **Kod Durumu:** `Reference`
```typescript
interface AIResponse<T> {
  data: T;
  confidence: number;
  reasoning?: string;
  sources?: string[];
}
```

### **🔥 Zod ile Validation Bridge**
> **Kod Durumu:** `Reference`
```typescript
import { z } from "zod";

const UserSchema = z.object({
  name: z.string(),
  age: z.number(),
});

type User = z.infer<typeof UserSchema>;

function validateAIResponse(input: unknown): User {
  return UserSchema.parse(input);
}
```

### **🧠 Neden Zorunlu?**
AI çıktıları güvenilir değildir.

Bu yüzden:
- Her output validate edilmeli
- Type system ile enforce edilmeli

**TypeScript burada sadece developer tool değil, AI güvenlik katmanıdır.**

---

## 🧠 **2. AST-Based Code Understanding**

### **📘 Code Context Interface**
> **Kod Durumu:** `Reference`
```typescript
interface CodeContext {
  imports: string[];
  functions: string[];
  types: string[];
  dependencies: string[];
}
```

### **🔍 TypeScript Compiler API**
> **Kod Durumu:** `Reference`
```typescript
import ts from "typescript";

const source = ts.createSourceFile(
  "file.ts",
  code,
  ts.ScriptTarget.Latest
);
```

### **🧠 Amaç:**
👉 AI sadece string değil
👉 Kodun yapısını anlar

---

## 🔗 **3. LSP Integration**

### **📘 LSP Features Interface**
> **Kod Durumu:** `Reference`
```typescript
interface LSPFeatures {
  goToDefinition: Function;
  findReferences: Function;
  getDiagnostics: Function;
}
```

### **🔄 AI + LSP Flow**
> **Kod Durumu:** `Reference`
```typescript
// AI → symbol bulur
// AI → type check yapar  
// AI → gerçek hata analizi yapar

class AIEnhancedLSP {
  async analyzeCode(code: string): Promise<CodeAnalysis> {
    const diagnostics = await this.lsp.getDiagnostics(code);
    const symbols = await this.lsp.findSymbols(code);
    
    return {
      diagnostics,
      symbols,
      aiSuggestions: await this.ai.analyze(diagnostics)
    };
  }
}
```

---

## 🔧 **4. AI Tool Calling Layer**

### **📘 Typed Tool Interface**
> **Kod Durumu:** `Reference`
```typescript
interface AITool<TInput, TOutput> {
  name: string;
  description: string;
  inputSchema: z.ZodSchema<TInput>;
  outputSchema: z.ZodSchema<TOutput>;
  execute: (input: TInput) => Promise<TOutput>;
}
```

### **🔧 Example Tool**
> **Kod Durumu:** `Reference`
```typescript
const fixBugTool: AITool<{filePath: string, error: string}, {fix: string, confidence: number}> = {
  name: "fixBug",
  description: "Fix TypeScript error",
  inputSchema: z.object({
    filePath: z.string(),
    error: z.string()
  }),
  outputSchema: z.object({
    fix: z.string(),
    confidence: z.number()
  }),
  execute: async (input) => {
    // TS + AST + AI birleşimi
    const diagnostics = await this.getDiagnostics(input.filePath);
    const fix = await this.ai.generateFix(diagnostics);
    return this.applyFix(fix);
  }
};
```

---

## 🧪 **5. Type-Safe Prompt Engineering**

### **📘 Typed Prompt Templates**
> **Kod Durumu:** `Production-Ready`
```typescript
interface PromptTemplate<TInput, TOutput> {
  build: (input: TInput) => string;
  parse: (output: string) => PromptExecutionResult<TOutput>;
}
```

### **🔄 Structured Output Enforcement**
> **Kod Durumu:** `Reference`
```typescript
interface PromptExecutionResult<T> {
  raw: string;
  parsed?: T;
  valid: boolean;
  parseError?: string;
}
```

### **🎯 Implementation**
> **Kod Durumu:** `Reference`
```typescript
class TypeSafePrompt<TInput, TOutput> implements PromptTemplate<TInput, TOutput> {
  constructor(
    private template: string,
    private inputSchema: z.ZodSchema<TInput>,
    private outputSchema: z.ZodSchema<TOutput>
  ) {}
  
  build(input: TInput): string {
    this.inputSchema.parse(input); // Validate input
    return this.template.replace(/\{\{(\w+)\}\}/g, (match, key) => {
      return String(input[key as keyof TInput]);
    });
  }
  
  parse(output: string): PromptExecutionResult<TOutput> {
    try {
      const parsed = JSON.parse(output);
      const validated = this.outputSchema.safeParse(parsed);
      
      return {
        raw: output,
        parsed: validated.success ? validated.data : undefined,
        valid: validated.success,
        parseError: validated.success ? undefined : validated.error.message
      };
    } catch (error) {
      return {
        raw: output,
        valid: false,
        parseError: `JSON parse error: ${error.message}`
      };
    }
  }
}
```

---

## 🔄 **6. Codegen Pipeline**

### **📘 Pipeline Interface**
> **Kod Durumu:** `Reference`
```typescript
interface CodegenPipeline {
  input: string;
  generate: boolean;
  validate: boolean;
  typeCheck: boolean;
  test?: boolean;
}
```

### **🔄 Complete Flow**
> **Kod Durumu:** `Reference`
```typescript
class AITypeScriptPipeline {
  async generateCode(input: string): Promise<GeneratedCode> {
    // prompt → AI output → schema parse → type check → AST verify → test → accept/reject
    const prompt = this.buildPrompt(input);
    const aiOutput = await this.ai.generate(prompt);
    const parsedCode = this.parseCode(aiOutput);
    
    if (this.pipeline.validate) {
      await this.validateSchema(parsedCode);
    }
    
    if (this.pipeline.typeCheck) {
      await this.typeCheck(parsedCode);
    }
    
    if (this.pipeline.test) {
      await this.runTests(parsedCode);
    }
    
    return parsedCode;
  }
}
```

---

## 🧪 **7. Type-Driven Validation**

### **📘 Double Security**
> **Kod Durumu:** `Reference`
```typescript
interface TypeValidator {
  runtimeCheck: boolean;
  compileCheck: boolean;
}
```

### **🔒 Validation Layers**
> **Kod Durumu:** `Reference`
```typescript
// Runtime → Zod
// Compile → TypeScript

class DualValidator {
  async validate<T>(
    data: unknown, 
    schema: z.ZodSchema<T>,
    code?: string
  ): Promise<ValidationResult<T>> {
    // Runtime validation
    if (this.typeValidator.runtimeCheck) {
      const runtimeResult = schema.safeParse(data);
      if (!runtimeResult.success) {
        return { success: false, error: runtimeResult.error };
      }
    }
    
    // Compile-time validation
    if (this.typeValidator.compileCheck && code) {
      const compileResult = await this.typeCheck(code);
      if (!compileResult.success) {
        return { success: false, error: compileResult.error };
      }
    }
    
    return { success: true, data };
  }
}
```

### **👉 Çift Güvenlik:**
- Runtime validation (Zod)
- Compile-time validation (TypeScript)

---

## 🧠 **8. AI Error Recovery (Failure Atlas ile Bağlantı)**

### **📘 Error Interface**
> **Kod Durumu:** `Reference`
```typescript
interface AIError {
  message: string;
  type: "type-error" | "runtime-error";
  fixSuggestion?: string;
}
```

### **🔄 Recovery Flow**
> **Kod Durumu:** `Reference`
```typescript
// TS error → classify → Failure Atlas → fix → validate

class AIErrorRecovery {
  async handleTypeError(error: ts.Diagnostic): Promise<FixResult> {
    // 1. Classify error
    const errorType = this.classifyError(error);
    
    // 2. Search Failure Atlas
    const pattern = await this.failureAtlas.search(errorType);
    
    // 3. Generate fix
    const fix = await this.ai.generateFix(pattern);
    
    // 4. Validate fix
    const validation = await this.validateFix(fix);
    
    return validation.success ? fix : null;
  }
}
```

---

## 🔗 **9. Sistem Mimarisi: TypeScript Control Plane**

### **🎯 Büyük Bağlantı**
> **Kod Durumu:** `Reference`
```
TypeScript Layer = Control Plane
```

### **🔄 Complete System**
> **Kod Durumu:** `Reference`
```
GGUF → execution
RAG → knowledge  
Failure Atlas → debugging
TypeScript → control + validation
```

### **🏗️ Integration Points**
> **Kod Durumu:** `Reference`
```typescript
interface SystemArchitecture {
  execution: "GGUF/LLM Runtime";
  knowledge: "RAG System";
  debugging: "Failure Pattern Atlas";
  control: "TypeScript Control Plane";
}
```

---

## ⚡ **10. AI Strict Mode**

### **📘 AI-Specific Strict Mode**
> **Kod Durumu:** `Reference`
```typescript
interface AIStrictMode {
  requireSchema: boolean;
  rejectUnknown: boolean;
  enforceTypes: boolean;
}
```

### **🔒 Implementation**
> **Kod Durumu:** `Reference`
```typescript
class AITypeScriptCompiler {
  constructor(private strictMode: AIStrictMode) {}
  
  async compile(code: string): Promise<CompilationResult> {
    if (this.strictMode.requireSchema) {
      this.ensureSchemaExists(code);
    }
    
    if (this.strictMode.rejectUnknown) {
      this.enableNoImplicitAny();
    }
    
    if (this.strictMode.enforceTypes) {
      this.enableStrictChecks();
    }
    
    return this.performCompilation(code);
  }
}
```

---

## 🔧 **11. Code Transformation Safety**

### **📘 Transformation Validation**
> **Kod Durumu:** `Reference`
```typescript
interface CodeTransformationCheck {
  changedSymbols: string[];
  importGraphSafe: boolean;
  publicContractChanged: boolean;
  requiresHumanApproval: boolean;
}
```

### **🔍 Safety Analysis**
> **Kod Durumu:** `Reference`
```typescript
class TransformationSafety {
  async analyzeTransformation(
    originalCode: string,
    transformedCode: string
  ): Promise<CodeTransformationCheck> {
    const originalAST = ts.createSourceFile("orig.ts", originalCode, ts.ScriptTarget.Latest);
    const transformedAST = ts.createSourceFile("trans.ts", transformedCode, ts.ScriptTarget.Latest);
    
    return {
      changedSymbols: this.findChangedSymbols(originalAST, transformedAST),
      importGraphSafe: this.validateImportGraph(originalAST, transformedAST),
      publicContractChanged: this.checkPublicAPI(originalAST, transformedAST),
      requiresHumanApproval: this.needsHumanReview(originalAST, transformedAST)
    };
  }
}
```

---

## 📊 **12. Complete AI Pipeline**

### **🔄 Visual Pipeline Flow**
> **Kod Durumu:** `Reference`
```
prompt → AI output → schema parse → type check → AST verify → test → accept/reject
```

### **🎯 Implementation**
> **Kod Durumu:** `Reference`
```typescript
class CompleteAIPipeline {
  async process(input: string): Promise<ProcessResult> {
    try {
      // 1. Build prompt
      const prompt = this.promptBuilder.build(input);
      
      // 2. Generate AI output
      const aiOutput = await this.ai.generate(prompt);
      
      // 3. Schema parse
      const parsed = this.promptTemplate.parse(aiOutput);
      if (!parsed.valid) {
        return { success: false, error: parsed.parseError };
      }
      
      // 4. Type check
      const typeCheck = await this.typeChecker.check(parsed.parsed);
      if (!typeCheck.success) {
        return { success: false, error: typeCheck.error };
      }
      
      // 5. AST verify
      const astCheck = await this.astValidator.verify(parsed.parsed);
      if (!astCheck.safe) {
        return { success: false, error: astCheck.error };
      }
      
      // 6. Test
      const testResult = await this.testRunner.run(parsed.parsed);
      if (!testResult.success) {
        return { success: false, error: testResult.error };
      }
      
      return { success: true, result: parsed.parsed };
      
    } catch (error) {
      return { success: false, error: error.message };
    }
  }
}
```

---

## 📊 **Upgrade Etkisi**

| Alan | Önce | Sonra |
|------|------|--------|
| TypeScript Knowledge | 9.5 | 9.5 |
| Practical Usage | 7 | 9.5 |
| AI Integration | 5 | 9.5 |
| System Design | 7 | 9.6 |

---

## 🎯 **NET SONUÇ**

Bu katmanları eklediğinde TypeScript dokümanı:

❌ "TypeScript nasıl çalışır" 
✅ "TypeScript ile AI sistemi nasıl kurulur"

**Artık bu doküman:**
- ❌ Language guide
- ✅ **AI Control Plane blueprint**
- ✅ **System integration guide**
- ✅ **Enterprise AI development manual**

**AI-Native TypeScript!** 🚀

---

## 18. ADVANCED VARIANCE & GENERIC CONSTRAINTS

### Covariance ve Contravariance Mastery

> **Kod Durumu:** `Reference`
```typescript
// Variance öğrenmenin temeli: type relationship açısından liskov substitution
// Covariance (out): Return types can be more specific (subclasses)
// Contravariance (in): Parameter types can be more general (superclasses)
// Invariance: Exact type match required

// ✅ Producer pattern - Covariant (out)
interface Producer<out T> {
  produce(): T;
}

// Subtype relationship preserved
const stringProducer: Producer<string> = {
  produce: () => "hello"
};

const anyProducer: Producer<any> = stringProducer; // ✅ Valid - covariant

// ✅ Consumer pattern - Contravariant (in) 
interface Consumer<in T> {
  consume(item: T): void;
}

// Reverse subtype relationship for parameters
const anyConsumer: Consumer<any> = {
  consume: (item: any) => console.log(item)
};

const stringConsumer: Consumer<string> = anyConsumer; // ✅ Valid - contravariant

// 🎯 Real-world example: Event handling
interface EventListener<T> {
  // Input contravariant: accepts T or any supertype
  on(event: T): void;
}

interface EventEmitter<T> {
  // Output covariant: emits T or any subtype
  emit(): T;
}

// Filtering framework - variance helps with correctness
type FilterPredicate<in T> = (item: T) => boolean;
type Mapper<in T, out U> = (item: T) => U;

function map<T, U>(
  items: T[],
  mapper: Mapper<T, U>
): U[] {
  return items.map(mapper);
}

// ✅ Works because Mapper is contravariant in T, covariant in U
const numToString = (n: number | string): string => String(n);
const result = map([1, 2, 3], numToString); // Numeric array works with broader mapper
```

### Generic Constraint Deep Dive

> **Kod Durumu:** `Reference`
```typescript
// 1. Keyof constraints - type-safe property access
interface Config {
  apiUrl: string;
  timeout: number;
  retries: number;
}

function getConfig<K extends keyof Config>(key: K): Config[K] {
  // Compile-time guarantee: key must be valid Config property
  return config[key];
}

// 2. Conditional type with constraints
type IsString<T> = T extends string ? true : false;
type IsAsync<T> = T extends Promise<any> ? true : false;

// 3. Distributive conditional types (union handling)
type Flatten<T> = T extends Array<infer U> ? U : T;

type Str = Flatten<string[]>; // string
type Num = Flatten<number>; // number
type Combined = Flatten<(string | number)[]>; // string | number

// ✅ Union distributive: applied to each union member
type GetPromiseType<T> = T extends Promise<infer U> ? U : never;
type A = GetPromiseType<Promise<string> | Promise<number>>; // string | number

// 4. Template literal constraints
type EmailDomain = `${string}@${string}.${string}`;
const email: EmailDomain = "user@example.com"; // ✅
// const invalid: EmailDomain = "invalid-email"; // ❌

// 5. Recursive generic constraints with depth limiting
type DeepPartial<T, Depth extends number = 5> = 
  Depth extends 0 ? T :
  T extends object ? {
    [K in keyof T]?: DeepPartial<T[K], [...Array<1>, Depth][number]>
  } : T;

interface Nested {
  level1: {
    level2: {
      level3: {
        value: string;
      }
    }
  }
}

type PartialNested = DeepPartial<Nested, 3>;
// Depth limited to prevent infinite types

// 6. Multiple constraint combinations
type ValidID<T> = T extends (string | number) 
  ? T extends string 
    ? T extends `id_${infer Rest}` 
      ? T 
      : never 
    : T extends number 
      ? T extends Positive 
        ? T 
        : never 
      : never 
  : never;

type Positive = number & { readonly __brand: 'positive' };

const validId: ValidID<"id123"> = "id123"; // ✅
// const invalid: ValidID<"123"> = "123"; // ❌
```

---

## 19. TYPESCRIPT COMPILER API & METAPROGRAMMING

### Program Inspection

> **Kod Durumu:** `Reference`
```typescript
import ts from 'typescript';

interface CompilerAnalysis {
  // Type information
  types: Map<string, ts.Type>;
  symbols: Map<string, ts.Symbol>;
  sourceFiles: ts.SourceFile[];
  
  // Diagnostics
  diagnostics: ts.Diagnostic[];
  
  // Architecture
  dependencyGraph: Map<string, string[]>;
  unusedSymbols: string[];
  complexityMetrics: ComplexityMetric[];
}

class TypeScriptAnalyzer {
  private program: ts.Program;
  private typeChecker: ts.TypeChecker;
  
  constructor(configPath: string) {
    const config = ts.readConfigFile(configPath, ts.sys.readFile);
    const parsedConfig = ts.parseJsonConfigFileContent(
      config.config,
      ts.sys,
      require('path').dirname(configPath)
    );
    
    this.program = ts.createProgram(
      parsedConfig.fileNames,
      parsedConfig.options
    );
    this.typeChecker = this.program.getTypeChecker();
  }
  
  analyze(): CompilerAnalysis {
    const analysis: CompilerAnalysis = {
      types: new Map(),
      symbols: new Map(),
      sourceFiles: this.program.getSourceFiles(),
      diagnostics: this.program.getSemanticDiagnostics(),
      dependencyGraph: new Map(),
      unusedSymbols: [],
      complexityMetrics: []
    };
    
    // Walk AST and collect information
    for (const sourceFile of this.program.getSourceFiles()) {
      if (sourceFile.isDeclarationFile) continue;
      
      ts.forEachChild(sourceFile, node => {
        this.visitNode(node, analysis);
      });
    }
    
    return analysis;
  }
  
  private visitNode(node: ts.Node, analysis: CompilerAnalysis): void {
    if (ts.isVariableDeclaration(node)) {
      const type = this.typeChecker.getTypeAtLocation(node);
      analysis.types.set(node.name!.getText(), type);
    }
    
    if (ts.isInterfaceDeclaration(node) || ts.isClassDeclaration(node)) {
      const symbol = this.typeChecker.getSymbolAtLocation(node.name!);
      if (symbol) {
        analysis.symbols.set(node.name!.getText(), symbol);
      }
    }
    
    // Complex calculation: cyclomatic complexity
    if (ts.isFunctionDeclaration(node) || ts.isMethodDeclaration(node)) {
      const complexity = this.calculateCyclomaticComplexity(node);
      analysis.complexityMetrics.push({
        name: node.name?.getText() || 'anonymous',
        complexity,
        isHighComplexity: complexity > 10
      });
    }
    
    ts.forEachChild(node, child => this.visitNode(child, analysis));
  }
  
  private calculateCyclomaticComplexity(node: ts.FunctionLike): number {
    let complexity = 1;
    
    const visit = (n: ts.Node) => {
      if (ts.isIfStatement(n) || ts.isWhileStatement(n) || ts.isDoStatement(n)) complexity++;
      if (ts.isBinaryExpression(n) && n.operatorToken.kind === ts.SyntaxKind.BarBarToken) complexity++;
      if (ts.isBinaryExpression(n) && n.operatorToken.kind === ts.SyntaxKind.AmpersandAmpersandToken) complexity++;
      if (ts.isCaseClause(n)) complexity++;
      if (ts.isCatchClause(n)) complexity++;
      
      ts.forEachChild(n, visit);
    };
    
    ts.forEachChild(node, visit);
    return complexity;
  }
  
  detectUnusedSymbols(): string[] {
    const used = new Set<string>();
    const defined = new Set<string>();
    
    for (const sourceFile of this.program.getSourceFiles()) {
      ts.forEachChild(sourceFile, node => {
        if (ts.isVariableDeclaration(node) || ts.isFunctionDeclaration(node)) {
          defined.add(node.name?.getText() || '');
        }
      });
    }
    
    // Analyze references
    const unused: string[] = [];
    for (const symbol of defined) {
      const references = this.program.getSourceFiles()
        .flatMap(f => ({
          file: f,
          matches: f.getText().match(new RegExp(`\\b${symbol}\\b`)) || []
        }))
        .filter(r => r.matches.length > 1) // More than just definition
        .length;
      
      if (references === 0) {
        unused.push(symbol);
      }
    }
    
    return unused;
  }
  
  generateDependencyGraph(): Map<string, string[]> {
    const graph = new Map<string, string[]>();
    
    for (const sourceFile of this.program.getSourceFiles()) {
      const fileName = sourceFile.fileName;
      graph.set(fileName, []);
      
      // Find imports
      ts.forEachChild(sourceFile, node => {
        if (ts.isImportDeclaration(node) && ts.isStringLiteralLike(node.moduleSpecifier)) {
          const importPath = node.moduleSpecifier.text;
          graph.get(fileName)!.push(importPath);
        }
      });
    }
    
    return graph;
  }
}

interface ComplexityMetric {
  name: string;
  complexity: number;
  isHighComplexity: boolean;
}
```

### Type-Level Testing & Validation

> **Kod Durumu:** `Reference`
```typescript
// Testing utility types at compile time
namespace TypeTests {
  // Helper: compile-time assertion
  type Expect<T extends true> = T;
  type Equal<X, Y> = 
    (<T>() => T extends X ? 1 : 2) extends 
    (<T>() => T extends Y ? 1 : 2) ? true : false;
  
  // Test cases
  type Test1 = Expect<Equal<string, string>>; // ✅
  // type Test2 = Expect<Equal<string, number>>; // ❌ Error
  
  // Complex type validation
  type Test3 = Expect<Equal<
    Readonly<{a: string}>,
    {readonly a: string}
  >>; // ✅
  
  // Utility type correctness
  type TestFlattenArray = Expect<Equal<
    Flatten<[1, [2, [3]]]>,
    1 | 2 | 3
  >>; // ✅
}

// Runtime type guard generation
function createTypeGuard<T>(validator: (value: unknown) => value is T) {
  return (value: unknown): value is T => validator(value);
}

const isString = createTypeGuard((v): v is string => typeof v === 'string');
const isNumber = createTypeGuard((v): v is number => typeof v === 'number');

// Type narrowing helper
function narrow<T, K extends T>(value: T, guard: (v: T) => v is K): K | null {
  return guard(value) ? value : null;
}
```

---

## 20. MIGRATION STRATEGIES - TanStack/Vue/Angular Patterns

### Type Migration Framework

> **Kod Durumu:** `Reference`
```typescript
// Strategy 1: Gradual adoption (most common)
// Phase 1: Enable checkJs but allow any
// Phase 2: Convert critical paths first
// Phase 3: Enable strict mode progressively

interface MigrationPhase {
  phase: number;
  tsconfig: Partial<ts.CompilerOptions>;
  files: string[];
  estimatedHours: number;
  risks: string[];
  rollbackStrategy: string;
}

const migrationPlan: MigrationPhase[] = [
  {
    phase: 1,
    tsconfig: {
      checkJs: true,
      allowJs: true,
      noImplicitAny: false
    },
    files: ['src/utils/**'],
    estimatedHours: 4,
    risks: ['Limited validation, some bugs hidden'],
    rollbackStrategy: 'Revert tsconfig.json'
  },
  {
    phase: 2,
    tsconfig: {
      noImplicitAny: true,
      strictNullChecks: false
    },
    files: ['src/api/**', 'src/types/**'],
    estimatedHours: 12,
    risks: ['Type errors may appear, review needed'],
    rollbackStrategy: 'Git stash + revert commit'
  },
  {
    phase: 3,
    tsconfig: {
      strict: true,
      strictNullChecks: true,
      strictFunctionTypes: true,
      strictPropertyInitialization: true
    },
    files: ['src/'],
    estimatedHours: 40,
    risks: ['Major refactoring needed'],
    rollbackStrategy: 'Create feature branch, merge gradually'
  }
];

// Strategy 2: Component-driven migration (Vue/React)
interface ComponentMigration {
  component: string;
  currentState: 'JavaScript' | 'JSDoc' | 'Partial TypeScript' | 'Full TypeScript';
  nextState: 'Full TypeScript';
  props: Record<string, string>; // prop names to types
  emits: Record<string, string>; // event names to signatures
  estimatedHours: number;
}

// Strategy 3: Monorepo migration
class MonorepoMigrationOrchestrator {
  async migrateDependencyOrder(packages: Package[]): Promise<void> {
    // Topological sort: no circular dependencies
    const sorted = this.topologicalSort(packages);
    
    for (const pkg of sorted) {
      console.log(`Migrating ${pkg.name}...`);
      await this.migratePackage(pkg);
      await this.runTests(pkg);
    }
  }
  
  private topologicalSort(packages: Package[]): Package[] {
    // Implementation: Kahn's algorithm
    const graph = new Map<string, string[]>();
    const inDegree = new Map<string, number>();
    
    for (const pkg of packages) {
      if (!inDegree.has(pkg.name)) {
        inDegree.set(pkg.name, 0);
        graph.set(pkg.name, []);
      }
      
      for (const dep of pkg.dependencies) {
        if (!graph.has(dep)) graph.set(dep, []);
        graph.get(dep)!.push(pkg.name);
        inDegree.set(pkg.name, (inDegree.get(pkg.name) || 0) + 1);
      }
    }
    
    const queue = [...packages.filter(p => inDegree.get(p.name) === 0)];
    const sorted: Package[] = [];
    
    while (queue.length > 0) {
      const current = queue.shift()!;
      sorted.push(current);
      
      for (const dependent of graph.get(current.name)!) {
        inDegree.set(dependent, inDegree.get(dependent)! - 1);
        if (inDegree.get(dependent) === 0) {
          const pkg = packages.find(p => p.name === dependent);
          if (pkg) queue.push(pkg);
        }
      }
    }
    
    return sorted;
  }
}

interface Package {
  name: string;
  dependencies: string[];
}
```

---

## 21. Sonuç: Type System Mastery

**Type System Engineering** sadece kod yazmak değil, bir **sistem mimarisi** kurmaktır:

- **Architectural Thinking**: Her pattern'in sistemdeki yerini anlamak
- **Performance Engineering**: Tip sisteminin compile-time performansını optimize etmek
- **Domain Modeling**: Business logic'i tip seviyesinde ifade etmek
- **Compiler API**: TypeScript compiler'ı kendi araçlarımız için kullanmak
- **Migration Strategy**: Gradual adoption ve large-scale refactoring
- **AI Integration**: Modern AI IDE'leri ile uyumlu kod yazmak

**Ultimate Goal:** TypeScript'i sadece bir dil olarak değil, bir **mimari araç, type-safe domain modeling ve AI işbirliği platformu** olarak kullanmak.

**Final Advice:** "En iyi tip mühendisi, en basit çözümü bulan, compiler internals'ı anlayan ve AI ile etkili işbirliği yapabilen mühendistir."

---

## 22. v1.4 Canonical Contract Alignment

TypeScript katmanı, cross-layer sözleşmelerini runtime'da zorlamak için aşağıdaki kapıları normatif kabul eder.

### Runtime Contract Gate

> **Kod Durumu:** `Reference`
```typescript
type LayerName = 'decision' | 'rag' | 'model' | 'safety' | 'tool' | 'verification';

interface RuntimeContractGate {
  validateInput(layer: LayerName, payload: unknown): ValidationResult;
  validateOutput(layer: LayerName, payload: unknown): ValidationResult;
}
```

Kural: Şema doğrulaması geçmeyen payload bir sonraki katmana gönderilemez.

### Contract Test Suite

> **Kod Durumu:** `Reference`
```typescript
interface ContractTestSuite {
  runSchemaCompatibilityTests(): Promise<TestReport>;
  runCrossLayerIntegrationTests(): Promise<TestReport>;
  runErrorPropagationTests(): Promise<TestReport>;
}
```

Kural: CI'da `schema`, `error taxonomy`, `idempotency` testleri tek paket halinde koşar.

---

## 23. v1.4 Runtime Enforcement Pipeline

### Cross-layer Validation Gateway

> **Kod Durumu:** `Reference`
```typescript
class ContractGateway {
  constructor(private readonly gate: RuntimeContractGate) {}

  async execute<TIn, TOut>(
    layer: LayerName,
    input: TIn,
    handler: (validatedInput: TIn) => Promise<TOut>
  ): Promise<TOut> {
    const inputValidation = this.gate.validateInput(layer, input);
    if (!inputValidation.success) throw new Error('E_VALIDATION:input');

    const output = await handler(input);
    const outputValidation = this.gate.validateOutput(layer, output);
    if (!outputValidation.success) throw new Error('E_VALIDATION:output');

    return output;
  }
}
```

### Contract Test Matrix (CI)

| Test Pack | Scope | Blocking |
|---|---|---|
| `schema-compat` | Backward compatibility + deprecation | Yes |
| `error-propagation` | `ContractErrorCode` uniformity | Yes |
| `idempotency-replay` | Same key -> same output semantics | Yes |

Kural: Bu üç test paketi green olmadan release promotion yapılamaz.

---

## 24. v1.5 Repo-Aware Code Intelligence Enforcement

### Semantic Diff + Edit Scope Guard

> **Kod Durumu:** `Reference`
```typescript
interface SemanticEditGuard {
  mapImpact(changedFiles: string[]): Promise<{ affectedSymbols: string[]; affectedPackages: string[] }>;
  validateTargeting(editPlan: EditPlan): Promise<{ safe: boolean; reason?: string }>;
}
```

### Test Impact Predictor

> **Kod Durumu:** `Reference`
```typescript
interface TestImpactPredictor {
  selectRequiredTests(changedSymbols: string[]): Promise<string[]>;
}
```

### Invariant Checker

Kural: `wrong edit target` veya `partial refactor breakage` riski varsa patch apply işlemi fail-closed durdurulur.

### Production Contract Enforcement Flow

> **Kod Durumu:** `Reference`
```typescript
interface ProductionContractFlow {
  preMerge: ['schema-compat', 'error-propagation', 'idempotency-replay'];
  canary: ['dto-canary', 'runtime-gate-canary'];
  postDeploy: ['replay-audit', 'drift-audit'];
}
```

Kural: Bu akışta herhangi bir gate kırılırsa release pipeline otomatik durur ve rollback tetiklenir.

### Fail-Closed Runtime Guard

> **Kod Durumu:** `Reference`
```typescript
class FailClosedRuntimeGuard {
  enforce(layer: LayerName, payload: unknown): void {
    const isValid = this.validate(layer, payload);
    if (!isValid) {
      throw new Error('E_VALIDATION:fail_closed');
    }
  }
}
```

Kural: Runtime guard üretimde opsiyonel değildir; disable edilemez.

### Core Execution Binding Contract

> **Kod Durumu:** `Reference`
```typescript
interface CoreExecutionBinding {
  requiredForCodeTasks: [
    'repoGraphBuild',
    'semanticDiffGuard',
    'editScopeValidation',
    'testImpactPrediction',
    'invariantCheck'
  ];
}
```

Kural: Bu bağlayıcı set, code task execution brain'in opsiyonel eklentisi değil zorunlu ön-koşuludur.

### Developer SDK Surface (Ergonomics)

> **Kod Durumu:** `Reference`
```typescript
interface DeveloperSDK {
  init(config: { mode: 'local' | 'staging' | 'production' }): Promise<void>;
  run(query: string, options?: { preset?: 'strict' | 'balanced' | 'cheap' }): Promise<ExecutionResult>;
  getTrace(traceId: string): Promise<{ summary: string; audits: string[] }>;
}
```

Kural: Yeni ekip üyeleri düşük seviyeli katmanları bilmeden `DeveloperSDK` ile ilk çalışır akışı kurabilmelidir.
