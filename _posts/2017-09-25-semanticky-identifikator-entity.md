---
layout: post
title:  Sémantický identifikátor entity"
categories: hibernate, java
---

Na projektech korporátních i startapových jsem se setkal vždy s tím, že si servisní třídy mezi sebou vyměnují identifikátory entit.

Často je to např. Long, pokud je datovým podkladem SQL databáze ale setkávám se i se Stringem, když je entita ze vzdáleného systému volaného přes MQ, WebServices, Rest apod.

Problém pak nastává v tom, že se tyto identifikátory často pletou.

Například

```java
public void updateTransaction(Long transactionId, 
                          Long userId, 
                          Long operatorId, 
                          Long clientId) {
    // ... do some nasty business logic
}
```

Není jasné, jestli userId je identifikátor entity User - nejspíš ano. A co operatorId? Je to taky z entity User? Obzvláště, když žádná entita Operator v doménovém modelu není. 


Co kdyby rozhraní vypadalo takto
```java
public void updateTransaction(TransactionId transactionId, 
                              UserId userId, 
                              UserId operatorId, 
                              ClientId clientId) {
    // ... do some nasty business logic
}
```

Takto už je sématický význam parametrů jasnější. Co ale ve skutečnosti obsahuje třída UserId?
```java
public final class UserId {
    private final Long value;
    
    private UserId(Long value) {
        this.value = value;
    }
    
    public Long getValue() {
        return value;
    }
    
    public static UserId of(Long value) {
        if (value==null) throw new IllegalArgumentException("Parameter 'value' can't be null.");  
        return new UserId(value); 
    }
    
    public static UserId of(String value) {
        if (value==null) throw new IllegalArgumentException("Parameter 'value' can't be null.");  
        return new UserId(Long.valueOf(value)); 
    }
}
```     

Instance je immutable a je tedy thread-safe. Zároveň hlídá, že hodnota value je ne-null-ová. Může obsahovat tolik factory metod kolik je potřeba - v tomto případě umí instanciovat z Long a ze String. 

Co když používám Hibernate? Jak to mám namapovat? Můžeme nentitu nechat tak jak je - anotovanou nad attributy - pak Hibernate mapuje hodnoty přímo na atributy bez použití set a get metod.

```java
@Entity
@Table(name = "COOL_USER")
public class User {
    @Id
    @GeneratedValue
    @Column(name = "USER_ID")
    private Long id;
    
    @Column(name = "FIRST_NAME")
    private String firstName;
    
    @Column(name = "LAST_NAME")
    private String lastName;
    
    public UserId getId() {
        return UserId.of(id);
    }
    
    public void setId(UserId id) {
        this.id = id.getValue();
    }
     
    // other set get methods for firstName and lastName
 
}
```

Čitelnost servisního kódy je mnohem lepší a navíd kompilátor za vás dělá kontrolu, jestli jste parametry nepopletli. 

Myslím si, že podobný přístup by se dal použít i na jiné jednoduché datové typy - např. AccountNumber, BirthNumber, DateOfBirth místo String, String, Date. 
