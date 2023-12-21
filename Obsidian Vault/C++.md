
<h1>Стараемся пользоваться вектором правильно!</h1>

1. reserve и emplace_back и shrink_to_fit. У стандартного вектора есть особенность, самовыделение памяти. Это и удобно и проблематично. Удобно потому что нам самим не нужно увеличивать размер нашего блока памяти, данный контейнер делает это за нас. Проблема в 2 вещах. 1. Как мы знаем выделение памяти с помощью оператора new или других си-стайл функций выделения памяти это системый вызов то есть Context Switch, когда наша программа останавливается и передаеться управление процессу операционной системы, которй дана задача выполнить вызов системного вызова (выделения памяти, создания потоков, синхронизация потоков и т.д.), после того так вызов был выполнен управление переходит снова в управление процессу программы. Это дорого и частые вызовы могут замедлить нашу программу. 2. Учитывая еще что вектор выделает постоянно слишком много памяти, когда размер (количество объектов в контейнере), становиться равным размеру выделенного блока памяти, он увеличивает размер выделенного блока памяти в 2 раза. то есть сначало был 1, потом 2, 4,8,16 и т.д. Из-за этого программисты могу писать не совсем эффективный код.

Все это решает три функции!!!

1. reserve, принимает размер, который надо зарезервировать, то есть выделить сразу. Это решает 1 проблему, частое выделение памяти. Так как, скорее всего, мы заранее знаем сколько элементов будет храниться в нашем контейнере, или выделить примерно нужный участок памяти, зачасту при большом размере это будет удачно, так как разрыв, перед новым выделение памяти в векторе, аж в 2 раза.

```cpp
    std::vector<int> data{};
    data.reserve(300);
    // или так
    std::vector<int> data_2{300};
```

Поэтому выделяйте память зарнее или резервируйте ее, это поможет сделать вашу программу более эффективной!

Далее emplace_back,emplace и emplace_front. Если наш вектор содержит тип, который весит достаточно много, к примеру такой же вектор, то вставка в него может быть дорогой, так как у нас происходит копирование объекта целиком, а копирование это плохо. Поэтому есть методы emplace! Метод с именем emplace принимает аргументы, которые должны передавать в конструктор объекта. Сам же объект создаеться внутри метода emplace, из-за чего копирование большого объекта не будет, конечно вы можете сказать что, если хранимый тип объекта принимает 4 параметра и по факту у нас 4 копирования вместо 1, то тут вы не совсем правы, так как здесь ситуация такова, что объект такого класса будет тяжелее копировать, нежели копирования его параметров.

```cpp
    #include <vector>
    #include <string>
    
    class Object {
        public:
        Object(std::string name) {
            name_ = name;
        }
        private:
        std::string name_;
    };
    
    int main() {
        std::vector<std::string> data{10};
        data.emplace_back("123"); // внутри создас объект класса Object, в который передаст нашу строку.
    }
```

методы с именем emplace аналогичны методам с именем push, только в методы с именем emplace надо передавать данный, для того чтобы создать объект внутри. Важное замечание, так как вектор внутри себя не очищает указатели во избежанея ошибок связанных с тем, что он может удалить существующий и нужный объект, указатели внутри с помощью оператора new он не создаст (умные указатели тоже, тупые походу...)

```cpp
std::vector<int*> data{};
data.reserve(100);
data.emplace_back(); // data[0] == nullptr

data.emplace_back(10); // Ошибка! Программа не скомпилируеться так как он попытаеться в int* 10, а ему нужен указатель.
```

В случае указателей не важно что вы будете использовать или методы emplace или push.

Теперь бонус это shrink_to_fit. Если вы выделили слишком много памяти или зарезервировали слишком много и в принципе уверены что больше туда вставлять ничего не будут то лучше лишнее просто очистить, дабы лишний раз не захламлять память. Это можно сделать с помощью метода shrink_to_fit.

```cpp
std::vector<int> data{300};
data.emplace_back(10); // Выделили памяти на 300 элементов а по итогу там только 1.
data.shrink_to_fit(); // Теперь память размером только под 1 элемент, все классно, Json Стетхэм одобряет
```


<h1>Передавайте в захват лямбды weak_pointer</h1>

Лямбда выражения очень сильно изменяют жизнь программиста позволяю улучать код и делая его более гибким, благодаря захватам мы можем получать что угодно и обновлять функционал.

Конечно же бывает нужда захватить указатель:

```cpp
int* ptr = new int{10};
auto handler = [ptr](){
	*ptr = 20;
};

handler();

std::cout << *ptr << "\n";

```

В данном коде есть 2 уязвимости:
1) Отсутствие безопасности:  В случае удаления указателя, мы никак не узнает это внутри нашей лямбды, в скажете да как так:
```cpp
int* ptr = new int{10};
auto handler = [p = &ptr](){ // Тяжело следить за такими указателями
	if (!p) {
		return;
	}

	*(*ptr) = 10;
}; 
delete ptr;
ptr = nullptr;
```
В этом случае мы в принципе получим наш результат однако проблема. Следить за указателем таким образом очень тяжело и к тому же при захвате указателя мы получаем утечку ^3^.  (в 1 случае, во 2 все норм).
2) Утечки:
Поэтому лучше использовать умные указатели но не shared_ptr, так как мы увеличиваем счетчик сильных  ссылок из-за чего получим утечку так как на наш указатель останется сильная ссылка, из-за чего он не удалиться.

```cpp
auto s_ptr = make_shared<int>(10); 
auto handler = [p = s_ptr](){ // утечка
	if (!p) {
		return;
	}

	*p = 10;
}; 
```
Осталась ссылка на p, утечка.

Поэтому нужно использовать слабые ссылки дабы не увеличивать счетчик ссылок, к тому же легко проверить указатель на валидность:
```cpp
auto s_ptr = make_shared<int>(10);
auto handler = [wp = weak_ptr<int>(s_ptr)](){ // Все в норме.
	if (wp.expired() {
		return;
	}

	*wp.lock() = 10;
};

```

<h1>map не всегда лучше if</h1>
Бывает такое когда надо проверить несколько условий и сделать различные действия в зависимости от условия

```cpp

enum class Type;

struct Props {
	using PropsType = optional<float>;
	PropsType hp;
	PropsType damage;
	PropsType speed;
	PropsType armor;
};

struct {
	operator optional<float>() const {return {0}; }
} _;

class Player {
private:
	using Handler = function<Proprs()>;
public:
	void Use(const Handler& handler) {
	
	}
private:
	Props _props;
	Type _type;
};

enum class Type {
	EFFECT,
	DEBUF,
	HEAL,
};

Proprs InvokeEffect() {
	return {{10},{5},{5},{5}};
}

Props InvokeHeal() {
return {{100},_,_,_};
}

Props InvokeDebuf() {
	return {_,{-5},{-5},{-5}};
}

int main() {
	Player player{};
	auto type = Type::EFFECT;

	if (type == Type::EFFECT) {
		player.Use(InvokeEffect);
	}

	if (type == Type::DEBUF) {
		player.Use(InvokeDebuf);
	}
	
	if (type == Type::HEAL) {
		player.Use(InvokeHeal);
	}
}

```

Вот такая не большая программа, в зависимости от enum передаем нужную функцию и используем тот или иной эффект. Кому-то не понравиться такой код из if и возможно напишет вот так:

```cpp
...

int main() {
	Player player{};
	auto type = Type::EFFECT;

	map<Type,function<Props()>> properties = {
		{Type::EFFECT,InvokeEffect},
		{Type::DEBUF,InvokeDebuf},
		{Type::HEAL,InvokeHeal}
	};

	player.Use(properties[type]);
}

```

Да код стал выглядить лучше но какой по факту ценой?

Каждый новый итем в map это новое выделение памяти. Так как это указатель мы занимает 8 байт на каждый элемент.

3 * 8 = 24байт.

так же учитываем что у нас есть фиктивный узел для перемещения итератора по факту у нас будет 4 таких ноды и того 32байта. В целом нам все равно на байтики, вопрос в том как часто мы будем создавать эти узлы так как операция выделения памяти это context switch, что в целом может замедлить вашу программу если очень часто использовать такие вещи.

Можете конечно опровергнуть что if тоже имеет не достаток в виде ожидания загрузки операции процессором (branch prediction).  Но это не такая сильная проблема нежели частое выделение памяти, к тому же хоть map и использует allocator, базовый стандартный allocator совсем никуда не годиться для сложных программ.

Поэтому надо держать баланс и обдумывать когда и что лучше. Поверьте, если для вам 5-7 конструкций if тяжело читать, то дальше хуже.
В целом как и мне и вам, все равно, map это красивое решение, но в с++ всегда стоит учитывать то или иное решение.


<h1>Небольшой совет: предпочитайте unordered_map, map</h1>

Многие как и в прошлом примере используют map а не unordered_map, из-за чего, если много элементов, происходит частая балансировка, что может излишне сказаться на производительности. Если вам не надо содержать элементы отсортированными то предпочитайте unordered_map нежели map, учитывая количество элементов. В общем и целом как и в прошлом примере используйте что считаете нужным, но всегда стоит понимать каким инструментом вы пользуетесь.

<h1>Фича #1</h1>
Допустим у нас есть вот такой заголовочный файл:

```cpp
// header

#pragma once

namespace VStore {

class VEngine {
		...
	public:
		void Engine();
		...
};

}

```

Мы не любим много писанины и постоянно писать префикс VStore не удобно, дабы это не писать можно сделать вот так:

```cpp
// source

namespace VStore {

void VEngine::Engine() {
...
}

}

```

Оборачиваем наш source code в namespace, таким образом мы пишем меньше писанины, и это хорошо :3

<h1>variant, optional, tuple - кто это?</h1>
Без прелюдий, по порядку и с примерами:

Varian - класс, позволяющий хранить достаточно большое количество разных типов данных,
но в определенный момент времени имеет только 1 состояние:

```cpp

// Создаем объект
variant<int,string> values{};

values = 10; // в данный момент имеет значение типа int, 10

cout << get<int>(values) << "\n"; // выведеться 10

values = "123"; // теперь состояние на string, int потерялся.


cout << get<string>(values) << "\n"; // выведеться 123

cout << get<int>(values) << "\n"; // ошибка! в данный момент времени хранит string.

```

Очень удобен, как динамический тип дарнных.

Так же имеется возможность проверить есть то или иное значение в текущий момент времени:

```cpp

// Создаем объект
variant<int,string> values{};

values = 10; // в данный момент имеет значение типа int, 10

cout << get<int>(values) << "\n"; // выведеться 10

values = "123"; // теперь состояние на string, int потерялся.


cout << get<string>(values) << "\n"; // выведеться 123

cout << get<int>(values) << "\n"; // ошибка! в данный момент времени хранит string.

if (holds_alternative<int>(values)) { // false, в текущий момент хранит string
	cout << get<int>(values) << "\n";
}

```


Вариант будет полезен если не нужно выбрасывать ошибку повторно в программе и резко прерывать ее таким образом. Можно сохранить эту ошибку, и проверять имеется ли ошибка:

```cpp


class AlertMessage {
...
	public:
		void Show();
};

// some function
variant<int,exception_ptr> result{};
try {
	AlertMessage alert{};
	alert.Show();
	result = 200;
} catch(...) {
	result = current_exception();
}

// over method

class Logger {
...
	public:
	static void Log(const exception_ptr&);
...
};

if (holds_alternatie<exception_ptr>(result)) {
	Logger::Log(get<exception_ptr>(result));
}

```

Так же из полезных функций имеется ***visit***!

visit  первым аргументом принимает функции а втором variant, visit, как сам перевод, позволит вам заглянуть в ваш variant и воспроизвести тот метод, к+ какому состоянию он относиться, сейчас покажу:

```c++
class Player {  
...
};  
  
enum class AttackType {  
  BASE,  
  KRIT,  
};  
  
enum class MagicAttackType {  
  VOID,  
  POISON  
};  
  
using AttackHandler = function<float(const Player&)>;  
  
struct Functions {  
  
  static float BaseDamage(const Player& player) {  
    cout << "BaseDamage\n";  
...
  }  
  static float KritDamage(const Player& player) {  
    cout << "KritDamage\n";  
...
  }  
  static float VoidDamage(const Player& player) {  
    cout << "VoidDamage\n";  
... 
  }  
  static float PoisonDamage(const Player& player) {  
    cout << "PoisonDamage\n";  
... 
  }};  
  
void InvokeAttack(const Player& lhs, Player& rhs, const AttackHandler& handler) {
...
  handler(lhs);
  ...
}  
  
template<class... Ts>  
struct overloaded : Ts... { using Ts::operator()...; };  
template<class... Ts>  
overloaded(Ts...) -> overloaded<Ts...>;  
  
using variant_attack = variant<AttackType, MagicAttackType>;  
  
int main() {  
  
  vector<variant_attack> attackTypes{  
      AttackType::KRIT,  
      AttackType::BASE,  
      MagicAttackType::VOID,  
      MagicAttackType::POISON  
  };  
  
  Player first{};  
  Player second{};  
  
  auto f_handler = [](AttackType type) -> AttackHandler   {  
    map<AttackType,AttackHandler> m_handler{  
        {AttackType::BASE, Functions::BaseDamage},  
        {AttackType::KRIT, Functions::KritDamage}  
    };  
    return m_handler[type];  
  };  auto s_handler = [](MagicAttackType type)  -> AttackHandler  {  
    map<MagicAttackType,AttackHandler> m_handler{  
        {MagicAttackType::VOID, Functions::VoidDamage},  
        {MagicAttackType::POISON, Functions::PoisonDamage}  
    };  
    return m_handler[type];  
  };  
  for(auto& item : attackTypes) {  
    auto handler = visit(overloaded{f_handler,s_handler}, item);  
    InvokeAttack(first,second,handler);  
  }  
}
```

Это пример кода который по типу атаки возвращает определенную функцию для вызова атаки,  каждая функция выводит свое имя. По итогу на экране мы сможем увидеть следующий текст:

```
KritDamage
BaseDamage
VoidDamage
PoisonDamage
```

visit позволяет, 