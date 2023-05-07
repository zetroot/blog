---
layout: post
title:  "Непростое простое сравнение"
date:   2023-05-02 12:30 +0300
categories: 
---

Как можно надурить голову себе, а затем QA на ровном месте? Тестировать не тот код, который выполняется приложением!

Это очень простая мораль, из немного более сложной истории, в которой случались моя самоуверенность, черезмерная вера в тесты, недостаточное понимание того, как работают методы BCL. Пришлось с этим разобраться. Собственно рассказ об этом разбирательстве последует ниже. 

# Завязка
Положим у нас стоит задача отслеживать изменения в выписке по счету, которую можно запрашивать из банка. Интересующая нас часть выписки представляет собой коллекцию транзакций по счету, где каждая транзакция обладает следующими атрибутами:
- Идентификатор получателя транзакции (куда собственно ушли деньги, может быть не заполнен, т.е. null)
- "Номер" транзакции - это устаревшее поле, гарантируется, что поле уникально в рамках одного получателя. Может быть незаполнен (для "новых" транзакций)
- Уникальный идентификатор транзакции - глобально уникальная строка, так же может быть не заполнена.

Конечно, есть еще различные атрибуты, типа даты, суммы и т.д., но для упрощения мы их рассматривать не будем. Далее будем работать со следующей моделью транзакции:
```cs
public record Transaction(int? TargetId, string? Uid, string? Account);
```

Отслеживать наш сервис должен появление новых транзакций, и оповещать пользователя о том, что добавилась новая транзакция.
При сравнении двух выписок, надо учитывать некоторые особенности. 
- Транзакция не может поменять получателя.
- Транзакция может пропасть из выписки
- Транзакция может появиться "задним числом" или изменить свою дату (поэтому дата транзакции не учитывается как ключевое поле)
- Произошло изменение формата: часть транзакций имеет только номер, часть имеет номер и уид, часть имеет только уид. 
- При этом присвоенный уид не может пропасть
- Присвоенный номер может пропасть, но только если он был заменен на уид
- Транзакция без получателя это какой то артефакт, о ее появлении репортить не надо.

В сухом остатке, задача выглядит достаточно просто: надо вычесть из множества транзакций текущей выписки множество транзакций предыдущей выписки и зарепортить все новые элементы. При этом сравнение транзакций проводить по некоторым нехитрым правилам. И кажется что для этого есть метод `IEnumerable<T>.Except<T>()` в linq, который принимает на вход "сравниватель": объект реализующий интерфейс `IEqualityComparer<T>` отвечающий на вопрос равенства двух объектов типа `T`.

# Пишем код
Сообразим незамысловатый сравниватель. Метод `Equals()` выглядит довольно прямолинейно:
```cs
public bool Equals(Transaction? x, Transaction? y)
{
    if (x == null || y == null) throw new ArgumentNullException(); // null не обрабатываем
    
    if (x.TargetId.HasValue && y.TargetId.HasValue && x.TargetId != y.TargetId) return false; // early exit: если получатель платежа заполнен для обоих транзакций и они не равны, то это явно разные транзакции.
    
    if (!string.IsNullOrEmpty(x.Uid) && !string.IsNullOrEmpty(y.Uid)) // сравниваем уникальные идентификаторы, только если они оба заполнены
    {
        return string.Equals(x.Uid, y.Uid, StringComparison.OrdinalIgnoreCase);
    }

    if (!string.IsNullOrEmpty(x.Account) && !string.IsNullOrEmpty(y.Account)) // то же для номеров счетов
    {
        return string.Equals(x.Account, y.Account, StringComparison.OrdinalIgnoreCase);
    }

    return false; // ничего не сработало, значит это разные транзакции
}
```

Получение хешкода сделаем также незамысловатым:
```cs
public int GetHashCode(Transaction obj) => 
    HashCode.Combine(
        obj.TargetId.GetHashCode(),
        obj.Uid?.GetHashCode(),
        obj.Account?.GetHashCode()
      );
```

# Пора приготовить немного тестов!

Сам тест прост как 3 рубля в тихую погоду:
```cs
[Test, TestCaseSource(typeof(TestData))]
public void Equals_WhenCalled_ReturnsExpectedValue(Transaction x, Transaction y, bool expected)
{
    //arrange
    var sut = new TransactionComparer();
    
    //act & assert
    sut.Equals(x, y).Should().Be(expected);
    sut.Equals(y, x).Should().Be(expected);
}
```

Вся магия в тестовых примерах:
```cs
[ExcludeFromCodeCoverage]
public class TestData : IEnumerable
{
    public IEnumerator GetEnumerator()
    {
        yield return new object[] { new Transaction(42, "uid", "account"),  new Transaction(69, "uid", "account"),  false };
        
        yield return new object[] { new Transaction(42, "uid", null),       new Transaction(42, "uid", null),       true  };
        yield return new object[] { new Transaction(42, "uid", ""),         new Transaction(42, "uid", null),       true  };
        yield return new object[] { new Transaction(42, "uid", "account"),  new Transaction(42, "uid", null),       true  };
        yield return new object[] { new Transaction(42, "uid", "acc1"),     new Transaction(42, "uid", "acc2"),     true  };
        
        yield return new object[] { new Transaction(42, "uid", null),       new Transaction(null, "uid", null),     true  };
        yield return new object[] { new Transaction(42, "uid", ""),         new Transaction(null, "uid", null),     true  };
        yield return new object[] { new Transaction(42, "uid", "account"),  new Transaction(null, "uid", null),     true  };
        yield return new object[] { new Transaction(42, "uid", "acc1"),     new Transaction(null, "uid", "acc2"),   true  };
        
        yield return new object[] { new Transaction(42, "uid", null),       new Transaction(42, "diu", null),       false  };
        yield return new object[] { new Transaction(42, "uid", ""),         new Transaction(42, "diu", null),       false  };
        yield return new object[] { new Transaction(42, "uid", "account"),  new Transaction(42, "diu", null),       false  };
        yield return new object[] { new Transaction(42, "uid", "acc1"),     new Transaction(42, "diu", "acc2"),     false  };
        
        yield return new object[] { new Transaction(42, "uid", null),       new Transaction(null, "olo", null),     false  };
        yield return new object[] { new Transaction(42, "uid", ""),         new Transaction(null, "olo", null),     false  };
        yield return new object[] { new Transaction(42, "uid", "account"),  new Transaction(null, "olo", null),     false  };
        yield return new object[] { new Transaction(42, "uid", "acc1"),     new Transaction(null, "olo", "acc2"),   false  };
        
        yield return new object[] { new Transaction(42, null, "acc"),       new Transaction(null, null, "acc"),     true   };
        yield return new object[] { new Transaction(42, "", "acc"),         new Transaction(null, null, "acc"),     true   };
        yield return new object[] { new Transaction(42, "uid", "acc"),      new Transaction(null, null, "caa"),     false  };
        yield return new object[] { new Transaction(42, "uid", "acc1"),     new Transaction(null, null, "acc1"),    true   };
        
        yield return new object[] { new Transaction(42, "uid", null),       new Transaction(42, null, "acc"),       false  };
        yield return new object[] { new Transaction(42, "uid", null),       new Transaction(null, null, "acc"),     false  };
    }
}
```
Чтож, этот набор тестов дает нам полное покрытие метода `Equals`. 
![Покрытие кода](/assets/equality-comparers/equals-coverage.png)
Теперь можно вычесть из одной коллекции другую при помощи `.Except()` и идти пить чай. Или нет? Ведь тестировщики приносят кучу примеров, когда код работает не так как ожидалось. Почему?

# А что мы тестировали то?
Реальный код, который использует компаратор немного отличается от тестового и выглядит примерно следующим образом:
```cs
var currentCollection = GetFromCurrentReport();
var previousCollection = _store.GetFromPreviousReport();
var diff = currentCollection.Except(previousCollection, sut);
``` 
Это немного сильно отличается от того что мы тестировали. Но кажется, что метод `Equals` там все равно используется. Что же пошло не так? 
Пора написать еще один тест, на тех же данных:
```cs
[Test, TestCaseSource(typeof(TestData))]
public void Except_WhenCalled_ReturnsEmptyCollection(Transaction x, Transaction y, bool isDiffEmpty)
{
    //arrange
    var sut = new TransactionComparer();
    var currentCollection = new[]{x};
    var previousCollection = new []{y};
    
    //act
    var diff = currentCollection.Except(previousCollection, sut).ToList();
    
    //assert
    if (isDiffEmpty)
        diff.Should().BeEmpty();
    else
        diff.Should().NotBeEmpty();
}
```
И... 10 из 23 тестов не проходят!
![Провал нового теста на тех же данных](/assets/equality-comparers/new-test-failure.png)
Но кое что изменилось, теперь появилось покрытие в методе `TransactionComparer.GetHashCode()`!
Пора заглянуть под капот linq-метода `Except()`

# Как вычитаются множества
Расчехляем декомпилятор или скачиваем символы отладки и начинаем проваливаться в реализацию `Except()`.
Собственно сам метод проверяет входные параметры и возвращает генератор:
```cs
private static IEnumerable<TSource> ExceptIterator<TSource>(
    IEnumerable<TSource> first,
    IEnumerable<TSource> second,
    IEqualityComparer<TSource> comparer)
{
    HashSet<TSource> set = new HashSet<TSource>(second, comparer);
    foreach (TSource source in first)
    {
        if (set.Add(source))
            yield return source;
    }
}
``` 
Что происходит? Из вычитаемой последовательности создается `HashSet`, в который в цикле напихиваются элементы исходной последовательности. Те, которые запихнуть удалось - возвращаются как элементы результирующей последовательности. We need to go deeper! Что происходит в вызове `HashSet.Add()`? Как хэшсет определяет новые элементы?
За полным кодом с разными хитрыми хаками можно сходить в [репозиторий рантайма](https://github.com/dotnet/runtime/blob/v7.0.5/src/libraries/System.Private.CoreLib/src/System/Collections/Generic/HashSet.cs#L1082), рассмотрим только основные пути выполнения:
```cs
private bool AddIfNotPresent(T value, out int location)
{
    /* Код подготовки */

    if (comparer == null)
    {
        /* Не наш случай - comparer предоставляется */
    }
    else
    {
        hashCode = value != null ? comparer.GetHashCode(value) : 0;
        bucket = ref GetBucketRef(hashCode);
        int i = bucket - 1; // Value in _buckets is 1-based
        while (i >= 0)
        {
            ref Entry entry = ref entries[i];
            if (entry.HashCode == hashCode && comparer.Equals(entry.Value, value))
            {
                location = i;
                return false; // <<== Выход в случае, если элемент не может быть добавлен
            }
            i = entry.Next;

            collisionCount++;
            /*Обаботка конкуретного обновления коллекции*/
        }
    }

    /*
    Бла бла бла, какой то сложный код, который обрабатывает добавление элемента в хэшсет
    */

    return true; // <<== А это выход, если элемент таки удалось добавить
}
```

Какой вывод можно сделать из увиденного? Чтобы новый элемент нельзя было добавить в хэшсет (т.к. там уже есть старый элемент) должно выполниться 2 условия: 
- хэшкоды этих элементов совпадают
- метод `Equals()` возвращает `true` для этой пары.

Очевидно, что реализация `GetHashCode()` использованная ранее генерирует хэшкоды неконсистетно с функцией сравнения, что и приводит к тому, что некоторые "одинаковые" элеметы можно добавить в один хэшсет.

# Исправляем баг
Перепишем функцию `GetHashCode()`, чтобы она всегда возвращала константу, тогда левая часть условия `entry.HashCode == hashCode && comparer.Equals(entry.Value, value)` всегда будет истинна, а собственно результат условия будет определяться только результатом `Equals()`.

```cs
public class PatchedTransactionComparer : TransactionComparer
{
    public override int GetHashCode(Transaction obj) => 0;
}
```
И вуа ля! Все тесты зеленые, баг исправлен.

# Скачать
Весь код доступен в [демо-репозитории под тегом `v1`](https://github.com/zetroot/DifficultSimpleCompare/tree/v1/tests)