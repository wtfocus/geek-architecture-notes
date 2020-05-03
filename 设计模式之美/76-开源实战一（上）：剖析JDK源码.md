[toc]

## 76 | 开源实战一（上）：剖析 Java JDK 源码

### 工厂模式在 Calender 类中的应用

-   Calendar 类提供了大量跟日期相关的功能代码，同时，又提供了一个 getInstance() 工厂方法，用来根据不同的 TimeZone 和 Locale 创建不同的 Calendar 子类对象。

-   Calendar 类的相关代码如下所示，下面只给出了 getInstance() 工厂方法的代码实现。

    -   ```java
        
        public abstract class Calendar implements Serializable, Cloneable, Comparable<Calendar> {
          //...
          public static Calendar getInstance(TimeZone zone, Locale aLocale){
            return createCalendar(zone, aLocale);
          }
        
          private static Calendar createCalendar(TimeZone zone,Locale aLocale) {
            CalendarProvider provider = LocaleProviderAdapter.getAdapter(
                CalendarProvider.class, aLocale).getCalendarProvider();
            if (provider != null) {
              try {
                return provider.getInstance(zone, aLocale);
              } catch (IllegalArgumentException iae) {
                // fall back to the default instantiation
              }
            }
        
            Calendar cal = null;
            if (aLocale.hasExtensions()) {
              String caltype = aLocale.getUnicodeLocaleType("ca");
              if (caltype != null) {
                switch (caltype) {
                  case "buddhist":
                    cal = new BuddhistCalendar(zone, aLocale);
                    break;
                  case "japanese":
                    cal = new JapaneseImperialCalendar(zone, aLocale);
                    break;
                  case "gregory":
                    cal = new GregorianCalendar(zone, aLocale);
                    break;
                }
              }
            }
            if (cal == null) {
              if (aLocale.getLanguage() == "th" && aLocale.getCountry() == "TH") {
                cal = new BuddhistCalendar(zone, aLocale);
              } else if (aLocale.getVariant() == "JP" && aLocale.getLanguage() == "ja" && aLocale.getCountry() == "JP") {
                cal = new JapaneseImperialCalendar(zone, aLocale);
              } else {
                cal = new GregorianCalendar(zone, aLocale);
              }
            }
            return cal;
          }
          //...
        }
        ```

    -   

### 建造者模式在 Calendar 类中的应用

-   建造者模式有两种实现方法：

    1.  一种是单独定义一个 Builder 类。
    2.  另一种是将 Builder 实现为原始类的内部类。

-   Calendar 就采用了第二种实现思路。

    -   ```java
        
        public abstract class Calendar implements Serializable, Cloneable, Comparable<Calendar> {
          //...
          public static class Builder {
            private static final int NFIELDS = FIELD_COUNT + 1;
            private static final int WEEK_YEAR = FIELD_COUNT;
            private long instant;
            private int[] fields;
            private int nextStamp;
            private int maxFieldIndex;
            private String type;
            private TimeZone zone;
            private boolean lenient = true;
            private Locale locale;
            private int firstDayOfWeek, minimalDaysInFirstWeek;
        
            public Builder() {}
            
            public Builder setInstant(long instant) {
                if (fields != null) {
                    throw new IllegalStateException();
                }
                this.instant = instant;
                nextStamp = COMPUTED;
                return this;
            }
            //...省略n多set()方法
            
            public Calendar build() {
              if (locale == null) {
                locale = Locale.getDefault();
              }
              if (zone == null) {
                zone = TimeZone.getDefault();
              }
              Calendar cal;
              if (type == null) {
                type = locale.getUnicodeLocaleType("ca");
              }
              if (type == null) {
                if (locale.getCountry() == "TH" && locale.getLanguage() == "th") {
                  type = "buddhist";
                } else {
                  type = "gregory";
                }
              }
              switch (type) {
                case "gregory":
                  cal = new GregorianCalendar(zone, locale, true);
                  break;
                case "iso8601":
                  GregorianCalendar gcal = new GregorianCalendar(zone, locale, true);
                  // make gcal a proleptic Gregorian
                  gcal.setGregorianChange(new Date(Long.MIN_VALUE));
                  // and week definition to be compatible with ISO 8601
                  setWeekDefinition(MONDAY, 4);
                  cal = gcal;
                  break;
                case "buddhist":
                  cal = new BuddhistCalendar(zone, locale);
                  cal.clear();
                  break;
                case "japanese":
                  cal = new JapaneseImperialCalendar(zone, locale, true);
                  break;
                default:
                  throw new IllegalArgumentException("unknown calendar type: " + type);
              }
              cal.setLenient(lenient);
              if (firstDayOfWeek != 0) {
                cal.setFirstDayOfWeek(firstDayOfWeek);
                cal.setMinimalDaysInFirstWeek(minimalDaysInFirstWeek);
              }
              if (isInstantSet()) {
                cal.setTimeInMillis(instant);
                cal.complete();
                return cal;
              }
        
              if (fields != null) {
                boolean weekDate = isSet(WEEK_YEAR) && fields[WEEK_YEAR] > fields[YEAR];
                if (weekDate && !cal.isWeekDateSupported()) {
                  throw new IllegalArgumentException("week date is unsupported by " + type);
                }
                for (int stamp = MINIMUM_USER_STAMP; stamp < nextStamp; stamp++) {
                  for (int index = 0; index <= maxFieldIndex; index++) {
                    if (fields[index] == stamp) {
                      cal.set(index, fields[NFIELDS + index]);
                      break;
                     }
                  }
                }
        
                if (weekDate) {
                  int weekOfYear = isSet(WEEK_OF_YEAR) ? fields[NFIELDS + WEEK_OF_YEAR] : 1;
                  int dayOfWeek = isSet(DAY_OF_WEEK) ? fields[NFIELDS + DAY_OF_WEEK] : cal.getFirstDayOfWeek();
                  cal.setWeekDate(fields[NFIELDS + WEEK_YEAR], weekOfYear, dayOfWeek);
                }
                cal.complete();
              }
              return cal;
            }
          }
        }
        ```

-   **思考**：既然已经有了 getInstance() 工厂方法来创建 Calendar 类对象，为什么还要用 Builder 来创建 Calendar 类对象呢？

    -   工厂模式与建造者模式的区别。
    -   **工厂模式**是用来创建不同，但是相关类型的对象（继承同一父类或者接口的了组子类），由给定的参数来决定创建哪种类型的对象。
    -   **建造得模式**用来创建一种类型的复杂对象，通过设置不同的可选参数，“定制化”地创建不同的对象。

-   上面代码中，后一半代码才属于标准和建造者模式，根据 setXXX() 方法设置参数，来定制化刚刚创建的 Calendar 子类对象。

### 装饰器模式在 Collections 类中的应用

-   Collections 类是一个集合窗口的工具类，提供了很多静态方法，用来创建各种集合窗口。

-   UnmodifiableCollection 类是 Collections 类的一个内部类，相关代码如下：

    -   ```java
        
        public class Collections {
          private Collections() {}
            
          public static <T> Collection<T> unmodifiableCollection(Collection<? extends T> c) {
            return new UnmodifiableCollection<>(c);
          }
        
          static class UnmodifiableCollection<E> implements Collection<E>,   Serializable {
            private static final long serialVersionUID = 1820017752578914078L;
            final Collection<? extends E> c;
        
            UnmodifiableCollection(Collection<? extends E> c) {
              if (c==null)
                throw new NullPointerException();
              this.c = c;
            }
        
            public int size()                   {return c.size();}
            public boolean isEmpty()            {return c.isEmpty();}
            public boolean contains(Object o)   {return c.contains(o);}
            public Object[] toArray()           {return c.toArray();}
            public <T> T[] toArray(T[] a)       {return c.toArray(a);}
            public String toString()            {return c.toString();}
        
            public Iterator<E> iterator() {
              return new Iterator<E>() {
                private final Iterator<? extends E> i = c.iterator();
        
                public boolean hasNext() {return i.hasNext();}
                public E next()          {return i.next();}
                public void remove() {
                  throw new UnsupportedOperationException();
                }
                @Override
                public void forEachRemaining(Consumer<? super E> action) {
                  // Use backing collection version
                  i.forEachRemaining(action);
                }
              };
            }
        
            public boolean add(E e) {
              throw new UnsupportedOperationException();
            }
            public boolean remove(Object o) {
               hrow new UnsupportedOperationException();
            }
            public boolean containsAll(Collection<?> coll) {
              return c.containsAll(coll);
            }
            public boolean addAll(Collection<? extends E> coll) {
              throw new UnsupportedOperationException();
            }
            public boolean removeAll(Collection<?> coll) {
              throw new UnsupportedOperationException();
            }
            public boolean retainAll(Collection<?> coll) {
              throw new UnsupportedOperationException();
            }
            public void clear() {
              throw new UnsupportedOperationException();
            }
        
            // Override default methods in Collection
            @Override
            public void forEach(Consumer<? super E> action) {
              c.forEach(action);
            }
            @Override
            public boolean removeIf(Predicate<? super E> filter) {
              throw new UnsupportedOperationException();
            }
            @SuppressWarnings("unchecked")
            @Override
            public Spliterator<E> spliterator() {
              return (Spliterator<E>)c.spliterator();
            }
            @SuppressWarnings("unchecked")
            @Override
            public Stream<E> stream() {
              return (Stream<E>)c.stream();
            }
            @SuppressWarnings("unchecked")
            @Override
            public Stream<E> parallelStream() {
              return (Stream<E>)c.parallelStream();
            }
          }
        }
        ```

-   如上代码中，最关键的一点是，**UnmodifiableCollection 的构造函数接收一个 Collection 类对象，然后对其所有的函数进行了包裹（Wrap）:重新实现（如 add() 函数）或者简单封装（如 stream() 函数）**。

### 适配器模式在 Collections 类中的应用

-   在新版本的 JDK 中，Enumeration 类是适配器类。

-   在代码实现角度，这个代码实现跟经典的适配器模式的代码实现，差别稍微有点大。

    -   enumeration() 静态函数的逻辑和 Enumeration 适配器类的代码耦合在一起。

    -   enumeration() 静态函数直接通过 new 的方式创建了匿名类对象。

    -   ```java
        
        /**
         * Returns an enumeration over the specified collection.  This provides
         * interoperability with legacy APIs that require an enumeration
         * as input.
         *
         * @param  <T> the class of the objects in the collection
         * @param c the collection for which an enumeration is to be returned.
         * @return an enumeration over the specified collection.
         * @see Enumeration
         */
        public static <T> Enumeration<T> enumeration(final Collection<T> c) {
          return new Enumeration<T>() {
            private final Iterator<T> i = c.iterator();
        
            public boolean hasMoreElements() {
              return i.hasNext();
            }
        
            public T nextElement() {
              return i.next();
            }
          };
        }
        ```

    -   

### 重点回顾

-   今天，我重点讲了，工厂模式、建造者模式、装饰器模式、适配器模式，这四种模式在 Java JDK 中的应用，主要目的是给你展示真实项目中是如何灵活应用设计模式的。
-   在真实项目开发中，这些模式的应用更加灵活，代码实现更加自由，可以根据具体的业务场景、功能需求，对代码实现做很大调整，甚至还可能会对模式本身的设计思路做调整。