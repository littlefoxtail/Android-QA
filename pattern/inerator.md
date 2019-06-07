# Iterator(迭代器模式)

提供一种方法顺序访问一个聚合对象中各个元素，而又不需要暴露该对象的内部表示。

迭代器模式的角色：

* Iterator（抽象迭代器）：它定义了访问和遍历元素的接口，声明了用于遍历数据元素的方法

    ```java
    public interface ItemIterator {
        boolean hasNext();

        Item next();
    }
    ```

* ConcreteIterator（具体迭代器）：它实现了抽象迭代器接口，完成了对聚合对象的遍历。同时在具体迭代器中通过游标来记录在聚合对象中的位置，在具体实现时，游标通常是一个表示位置的非负整数

    ```java
    public class TreasureChestItemIterator implements ItemIterator {
        private TreasureChest chest;
        private int idx;
        private ItemType type;

        public TreasureChestItemIterator(TreasureChest chest, ItemType type) {
            this.chest = chest;
            this.type = type;
            this.ldx = -1;
        }

        @Override
        public boolean hasNext() {
            return findNextIdx() != 1;
        }

        @Override
        public Item next() {
            ldx = findNextIdx();
            if (ldx != -1) {
                return chest.getItems().get(ldx);
            }
            return null;
        }

        private int findNextIdx() {
            List<Item> items = chest.getItems();
            boolean found = false;
            int tempIdx = idx;
            while(!found) {
                tempIdx++;
                if (tempIdx >= item.size()) {
                    tempIdx = -1;
                    break;
                }
                if (type.equals(ItemType.ANY) || items.get(tempIdx).getType.equals(type)) {
                    break;
                }
            }
            return tempIdx;
        }
    }
    ```

* Aggregate(抽象聚合类)：它用于存储和管理元素对象，声明一个creteIterator()方法用来创建一个迭代器对象，充当抽象迭代器工厂角色

* ConcreteAggregate（具体聚合类）：它实现了在抽象聚合类中声明的createIterator()方法，该方法返回一个与该具体聚合类对应的具体迭代器ConcreteIterator实例

    ```java
    public class TreasureChest {
        private List<Item> items;

        public TreasureChest() {
            items = new ArrayList<>();
            items.add(new Item(ItemType.POTION, "Potion of courage"));
            items.add(new Item(ItemType.RING, "Ring of shadows"));
            items.add(new Item(ItemType.POTION, "Potion of wisdom"));
            items.add(new Item(ItemType.POTION, "Potion of blood"));
            items.add(new Item(ItemType.WEAPON, "Sword of silver +1"));
            items.add(new Item(ItemType.POTION, "Potion of rust"));
            items.add(new Item(ItemType.POTION, "Potion of healing"));
            items.add(new Item(ItemType.RING, "Ring of armor"));
            items.add(new Item(ItemType.WEAPON, "Steel halberd"));
            items.add(new Item(ItemType.WEAPON, "Dagger of poison"));
        }

        ItemIterator iterator(ItemType itemType) {
            return new TreasureChestItemIterator(this, itemType);
        }

        public List<Item> getItems() {
            List<Item> list = new ArrayList<>();
            list.addAll(items);
            return list;
        }
    }
    ```

迭代器模式是一种使用频率非常高的设计模式，通过引入迭代器可以将数据遍历功能从聚合对象中分离出来，聚合对象只负责存储数据，而遍历数据由迭代器来完成

## 迭代器模式优点

1. 它支持不同方式遍历一个聚合对象，在同一个聚合对象上可以定义多种遍历方式。在迭代器模式中只需要用一个不同的迭代器来替换原有迭代器即可改变算法，可以自己定义迭代器的子类以支持新的遍历方式
2. 迭代器简化了聚合类。
3. 在迭代器模式中，由于引入了抽象层，增加新的聚合类和迭代器类都很方便，无须修改原有代码，满足“开闭原则”的要求

## 迭代器模式缺点

1. 由于迭代器模式将数据存储和遍历数据的职责分离，增加新的聚合类需要对应增加新的迭代器类，类的个数成对增加
2. 抽象迭代器的设计难度较大，需要充分考虑到系统将来的扩展

## 迭代器模式适应场景

1. 访问一个聚合对象的内容而无需暴露它的内部表示。将聚合对象的访问与内部数据的存储分离，使得访问聚合对象时无需了解其内部实现细节
2. 为一个聚合对象提供多种遍历。
3. 为遍历不同的聚合结构提供一个统一的接口(即支持多态迭代)
