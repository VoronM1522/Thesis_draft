# Исследование и адаптация компонентов персистентной Phantom ОС на Genode

# Research and adaptation of components from the persistent OS Phantom for Genode





**EUMEL** - персистентная ОСь 1979 года разработки

Нужно сказать про отличия от гибернации

The advantage of orthogonal persistence environments is simpler and less error-prone programs. (https://en.wikipedia.org/wiki/Persistence_(computer_science)#Orthogonal_or_transparent_persistence)

https://en.wikipedia.org/wiki/Capability-based_security





## Abstract

- **ru**

  ```
  Эволюция операционных систем на протяжении десятилетий определялась доказавшей свою жизнеспособность концептуальной моделью, сформировавшейся в середине XX века с появлением UNIX. Однако сегодня она всё чаще демонстрирует фундаментальные ограничения. Попытки обойти их еще в 80-х годах XX века привели в том числе и к появлению концепции операционных систем с персистентной памятью. Несмотря на достоинства такой парадигмы исследования фактически были прекращены, так как столкнулись с аппаратными ограничениями. С тех пор производительность компьютеров сильно выросла, что дало толчок новым работам в этом направлении. Яркой иллюстрацией этого является ОС Phantom. Однако написание и внедрение полностью новой операционной системы весьма длительно, трудоемко и влечет за собой большое количество ошибок. Смягчить эти последствия помогает использование уже отработанных механизмов, чему способствует портирование на Genode. В данной работе мы рассматриваем процесс адаптации компонентов PhantomOS на Genode. Мы описываем трудности, с которыми приходится сталкиваться, недостатки ОС и рассказываем о решениях, которые помогают избавиться от этих проблем или смягчить их.
  ```

  

- **en**

  ```
  The evolution of operating systems over the decades has been shaped by a proven conceptual model that emerged in the mid-20th century with the advent of UNIX. Today, however, it is increasingly revealing fundamental limitations. Attempts to circumvent these limitations as early as the 1980s led, among other things, to the emergence of the concept of operating systems with persistent memory. Despite the merits of this paradigm, research was effectively halted as it ran into hardware limitations. Since then, computer performance has increased significantly, which has spurred new work in this direction. A striking illustration of this is the Phantom OS. However, writing and implementing a completely new operating system is a very time-consuming and labor-intensive process and is prone to a large number of errors. The use of proven mechanisms helps mitigate these consequences, a process facilitated by porting to Genode. In this paper, we examine the process of adapting PhantomOS components to Genode. We describe the difficulties encountered, the OS’s shortcomings, and discuss solutions that help eliminate or mitigate these problems.
  ```

## Introduction

*введение (актуальность темы, цель, задачи, объект исследования, новизна темы исследования)*

- актуальность
  не обеспечена безопасность снапшотов в покое, любой человек, получающий доступ к снимку, получает аозможность воспроизвести состояние и работу системы и т. д.

- цель

  Обеспечить конфиденциальность персистентного состояния PhantomOS (путём интеграции механизма шифрования снапшотов)

- задачи

  - построить модель (схему) сценариев использования компьютера
  - выделить безопасные и небезопасные сценарии и указать те (тот), что планируем поддерживать
  - реализовать сценарий (адаптацией компонентов, написанием своего или как-либо еще)
  - провести тестирование (сравнительный анализ)
    - проверка доступности данных
    - сравнение скорости

- объект исследования
  безопасность снапшотов PhantomOS в покое

- новизна
  тема малоисследована, в контексте PhantomOS работ вообще не проводилось



### Background

- **ru**

  ```
  На сегодняшний активно используется концепция, при которой данные разделяются на используемые и хранимые. При этом ответственность за их перемещение между этими состояниями лежит на программисте. В контексте данной работы такой подход будет называться two-level store. К очевидным недостаткам такого подхода можно отнести:
  - Утрата данных неожиданном выключении/завершении процесса
  - Повышенная трудоемкость (и большая стоимость) разработки 
  Такому подходу противопоставляется концепция ортогональной персистентности, рассмотренная Аткинсоном и Морисоном в 90-х годах XX века (https://www.researchgate.net/publication/240720509_Persistent_Languages_and_Architectures). Он заключается в том, что к данным имеется постоянный доступ и не требуется явных действий со стороны программиста для их сохранения на диск. Вместо этого ответственность за это ложится на среду, в которой программа исполняется. При снижении количества манипуляций с данными, а они обычно занимают до 30% кода (https://www.researchgate.net/publication/240720509_Persistent_Languages_and_Architectures), очевидным образом снижается и количество ошибок при написании таких программ. К неочевидным приемуществам такого подхода можно отнести весьма эффективное использование пропускной способности диска (https://www.cs.utexas.edu/~lorenzo/corsi/439/ref/keykos.pdf). **(?)** За прошедшее время среди операционных систмем появились представители, реализующие эту концепцию. К таким можно отнести KeyKOS (EROS and Coyotos), Multics и другие **(НАЙТИ ССЫЛКИ И ПРОВЕРИТЬ)**. Все эти системы довольно старые, а после их появления исследования приостановились. Однако значительно выросший уровень аппаратного обеспечения вновь подтолкнул к исследованиям в этом направлении, следствием чего стало появление на свет операционной системы Aurora (transparent персистентность). К таким современным системам, реализующим концепцию ортогональной персистентности, относится проект PhantomOS.
  ```

- **en**

  ```
  Today, a widely used concept involves dividing data into two categories: active and stored. In this model, the programmer is responsible for moving data between these states. In the context of this paper, this approach will be referred to as a “two-level store.” The obvious drawbacks of this approach include:
  \begin{itemize}
      \item Data loss due to unexpected shutdowns or process termination;
      \item Increased development effort (and higher cost);
  \end{itemize}
  This approach is contrasted with the concept of orthogonal persistence, discussed by Atkinson and Morrison in the 1990 \cite{PLA}. It entails that data is constantly accessible and does not require explicit actions on the part of the programmer to save it to the drive. Instead, the responsibility for this falls on the environment in which the program is executed. This reduces the amount of data manipulation, which typically accounts for up to 30\% of the code \cite{PLA}. The number of errors when writing such programs is also significantly reduced. One of the less obvious advantages of this approach is the highly efficient use of disk bandwidth \cite{KOS}. Over time, several operating systems have emerged that implement this concept. These include KeyKOS (EROS and Coyotos), Multics (with single-level storage), and others. All of these systems are quite old, and research in this area came to a standstill after their introduction. However, significant advances in hardware capabilities have reignited interest in this field, leading to the development of the Aurora operating system (transparent persistence) \cite{Aurora}. The PhantomOS project is one such modern system that implements the concept of orthogonal persistence.
  ```

### PhantomOS

- **ru**

  ```
  Как заявляют сами разработчики (http://phantomos.org/ , https://phantomdox.readthedocs.io/en/latest/), Phantom OS - операционная система, реализующая принципы ортогональной персистентности внутри виртуальной машины Phantom (PVM). Как заявляется в документации, система гарантирует восстановление даже при нештатной перезагрузке с не слишком старыми консистентными данными. Достигается это засчет использования механизма снапшотов: время от времени система сбрасывает состояние PVM на диск, не останавливая при этом ее работу, что достигается использованием copy-on-write (CoW).
  На данный момент Phantom представляет из себя (proof of concept) PoC ортогональной персистентности. Многие механизмы и подсистемы реализованы, их работоспособность в некоторой степени проверена, но не протестирована доконца. Нынешний уровень стабильности не позволяет использовать ос в промышленных целях, и работы по ее доводке все еще ведутся.
  Одним из направлений таких работ является портирование PhantomOS на фреймворк Genode. Это должно ускорить разработку и помочь избежать большого количества ошибок путем использования уже проверенных решений, а также благоприятно сказаться на безопасности системы блягодаря улучшенной изоляции, что  обусловлено использованием capability-based безопасности.
  ```

- **en**

  ```
  According to the developers themselves, Phantom OS is an operating system that implements the principles of orthogonal persistence within the Phantom Virtual Machine (PVM) \cite{Phantom_docs}. As stated in the documentation, the system guarantees recovery (upon reboot, even an unexpected one) with consistent data that is not too old \cite{Phantom_docs}. This is achieved through the use of a snapshot mechanism. 
  At present, Phantom is a proof-of-concept (PoC) for the concept of orthogonal persistence. Many mechanisms and subsystems have been implemented, and their functionality has been tested to some extent; however, the current level of stability does not allow the OS to be used for industrial purposes, and work to improve it is still ongoing.
  One area of this work involves porting PhantomOS to the Genode framework. This should accelerate development and help avoid a large number of errors by using proven solutions, as well as positively impact system security through improved isolation, which is enabled by the use of capability-based security.
  ```

### Genode

- **ru**

  ```
  Genode OS Framework — это набор инструментов для построения операционных систем. Он предоставляет модель, в которой ресурсы компонентов изолируются, а их взаимодействие строится через RPC. Фреймворк поддерживает несколько ядер в качестве основы (в том числе NOVA, формально верифицированное seL4, Fiasco.OC, linux), предоставляет готовый набор драйверов, файловых систем и сервисов, что позволяет сосредоточиться на прикладной логике, не изобретая или перенося базовую инфраструктуру заново.
  Доступ к ресурсам в Genode организован в соответствии с принципами capability-based безопасности.
  В данном случае capability — это токен, дающий право на выполнение конкретной операции над конкретным объектом. Компонент может воспользоваться ресурсом только при наличии соответствующей capability, которую ему явно передал родительский компонент. При этом родительский компонент полностью ответственен за предоставление соответствующих capabilities дочернему и за передачу его запросов далее по иерархии. Это исключает возможность несанкционированного доступа к ресурсам в обход иерархии, что значительно отличается от подхода привычных нам сегодня ОС, в которых процесс с достаточными привилегиями может обратиться к произвольному ресурсу.
  В рамках проекта phantomuserland-snapper **(Ссылка)** PhantomOS портируется на Genode в виде набора компонентов. Виртуальная машина Phantom (PVM) исполняется как пользовательский процесс Genode.  Для создания снапшотов, управления ими и восстановления состояния из был разработан snapper **(Ссылка)**, использующий ФС для их хранения. Такой подход позволяет Phantom пользоваться проверенными драйверами и сервисами Genode, не реализуя их самостоятельно, изоляция компонентов Genode обеспечивает дополнительный рубеж защиты, а использование микроядер позволяет добиться скромных, по сегодняшним меркам, размеров trusted computing base (TCB), что также положительно сказывается на безопасности ОС.
  ```

- **en**

  ```
  The Genode OS Framework is a set of tools for building operating systems. It provides a model in which component resources are isolated and their interactions are mediated via RPC. The framework supports multiple kernels as a foundation (including NOVA, formally verified seL4 \cite{seL4}, Fiasco.OC, and Linux), and provides a ready-made set of drivers, file systems, and services, allowing developers to focus on application logic without having to reinvent or port the underlying infrastructure.
  Access to resources in Genode is organized in accordance with the principles of capability-based security.
  In this context, a capability is a token granting the right to perform a specific operation on a specific object. A component can use a resource only if it possesses the corresponding capability, which was explicitly granted to it by a parent component. At the same time, the parent component is fully responsible for granting the appropriate capabilities to the child component and for passing its requests further up the hierarchy. This prevents unauthorized access to resources by bypassing the hierarchy, which differs significantly from the approach of the operating systems we are familiar with today, where a process with sufficient privileges can access any resource. \cite{Genode_Foundations}.
  As part of the phantomuserland-snapper project \cite{GitHub_phantomuserland-snapper}, PhantomOS is being ported to Genode as a set of components. The Phantom Virtual Machine (PVM) runs as a Genode user process.  To create, manage, and restore snapshots, snapper \cite{GitHub_Snapper} was developed, which uses the file system for storage. This approach allows Phantom to use Genode’s proven drivers and services without implementing them independently; the isolation of Genode components provides an additional layer of protection; and the use of microkernels enables a trusted computing base (TCB) of modest size by today’s standards, which also positively impacts the OS’s security.
  ```

### Problem statement

- **ru**

  ```
  Однако некоторые проблемы безопасности все еще предстоит решить. Так, в отличие от большинства современных ОС, которые позволяют выполнять шифрование дискового пространства, PhantomOS не поддерживает это ни в каком виде. Поиск и анализ исследований также показал, что работы с PhantomOS в этом направлении не велись. Портирование также не решает проблему, так как в Genode нет готовых адаптированных механизмов шифрования диска. При том информация, сбрасываемая на диск в момент создания снапшота и хранящаяся там, может быть более чувствительной, нежели файлы на диске, поскольку в момент снимка в памяти могут оказаться конфиденциальные данные. Это во многом схоже с проблемой безопасности снапшоов виртуальных машин. При определенных условиях это может позволить полностью воспроизвести состояние и дальнейшую работу машины на устройстве злоумышленника. Из этого вытекает необходимость реализации защиты снапшотов в покое.
  Таким образом, целью работы является обеспечение конфиденциальности снапшотов PhantomOS в покое.
  Из этого вытекают следующие цели:
  - Проанализировать сценарии использования компьютера и выделить среди них безопасные сценарии, которые будут поддерживаться нами
  - Разработать или адаптировать компонент Genode, реализующий такой сценарий
  - Верифицировать корректность защиты: продемонстрировать недоступность содержимого снапшота
  - Оценить влияние шифрования на производительность путём сравнительного анализа скорости работы с включённым и отключённым шифрованием
  
  
  ```

- **en**

  ```
  However, some security issues still need to be addressed. For instance, unlike most modern operating systems, which support disk encryption, PhantomOS does not support this feature in any form. A review of the literature also revealed that no research has been conducted on PhantomOS in this area. Porting does not solve the problem either, as Genode lacks ready-made, adapted disk encryption mechanisms. Moreover, the information flushed to disk at the moment a snapshot is created and stored there may be more sensitive than the files on the disk, since confidential data may be present in memory at the time the snapshot is taken. This is largely similar to the security issue with virtual machine snapshots. Under certain conditions, this could allow the state and subsequent operation of the machine to be fully reproduced on an attacker’s device. This necessitates the implementation of protection for snapshots at rest.
  Thus, the goal of this work is to ensure the confidentiality of PhantomOS snapshots at rest.
  This leads to the following objectives:
  \begin{itemize}
      \item Analyze computer usage scenarios and identify among them the secure scenarios that we will support;
      \item Develop or adapt a Genode component that implements such a scenario;
      \item Verify the correctness of the protection: demonstrate that the snapshot’s contents are inaccessible;
      \item Evaluate the impact of encryption on performance by comparing the speed of operations with encryption enabled and disabled;
  \end{itemize}
  ```

### Background

1. Каково текущее положение дел (с памятью)?
   На сегодняшний активно используется концепция, при которой данные разделяются на используемые и хранимые. При этом ответственность за их перемещение между этими состояниями лежит на программисте. В контексте данной работы такой подход будет называться two-level store. К очевидным недостаткам такого подхода можно отнести:

   - Утрата данных неожиданном выключении/завершении процесса
   - Повышенная трудоемкость (и большая стоимость) разработки 
   - **Дописать недостатки Two-level store**

   Такому подходу противопоставляется концепция ортогональной персистентности, рассмотренная Аткинсоном и Морисоном в 80-х годах XX века (https://www.researchgate.net/publication/240720509_Persistent_Languages_and_Architectures).

2. Что за концепция ((ортогональной) персистентности)? Когда зародилась и кем сформулирована? В чем заключается? **(Надо знать отличие ортогональной от неортогональной)**
   Он заключается в том, что к данным имеется постоянный доступ и не требуется явных действий со стороны программиста для их сохранения на диск. Вместо этого ответственность за это ложится на среду, в которой программа исполняется. При снижении количества манипуляций с данными, а они обычно занимают до 30% кода (https://www.researchgate.net/publication/240720509_Persistent_Languages_and_Architectures), очевидным образом снижается и количество ошибок при написании таких программ.

3.  Какие приемущества?

   К неочевидным приемуществам такого подхода можно отнести весьма эффективное использование пропускной способности диска (https://www.cs.utexas.edu/~lorenzo/corsi/439/ref/keykos.pdf, требует пояснений). **(?)**

4. Какие есть представители?
   За прошедшее время среди операционных систмем появились представители, реализующие эту концепцию. К таким можно отнести KeyKOS (EROS and Coyotos), Multics **(нет, но тут реализована концепция single-level памяти (https://ru.wikipedia.org/wiki/Multics). Можно дополнить информацией о такой концепции (https://ru.wikipedia.org/wiki/%D0%9E%D0%B4%D0%BD%D0%BE%D1%83%D1%80%D0%BE%D0%B2%D0%BD%D0%B5%D0%B2%D0%BE%D0%B5_%D1%85%D1%80%D0%B0%D0%BD%D0%B8%D0%BB%D0%B8%D1%89%D0%B5).)** и другие **(НАЙТИ ССЫЛКИ И ПРОВЕРИТЬ)**, однако на сегодняшний день самой современной и активно развивающейся является OS Phantom.

### PhantomOS

1. Что такое PhantomOS?

   Как заявляют сами разработчики (http://phantomos.org/ , https://phantomdox.readthedocs.io/en/latest/), Phantom OS - операционная система, реализующая принципы ортогональной персистентности внутри виртуальной машины Phantom (PVM). Как заявляется в документации, система гарантирует восстановление (при перезагрузке, даже неожиданной) с не слишком старыми консистентными данными. Достигается это засчет использования механизма снапшотов. 

2. На каком этапе развития?
   На данный момент Phantom представляет из себя PoC концепта ортогональной персистентности. Многие механизмы и подсистемы реализованы, их работоспособность в некоторой степени проверена, однако нынешний уровень стабильности не позволяет использовать ос в промышленных целях, и работы по ее улучшению все еще ведутся.

3. Зачем нужно портирование на Genode?
   Одним из направлений таких работ является портирование PhantomOS на фреймворк Genode. Это должно ускорить разработку и помочь избежать большого количества ошибок путем использования уже проверенных решений, а также благоприятно сказаться на безопасности системы блягодаря улучшенной изоляции, что  обусловлено использованием capability-based безопасности.

### Genode

1. Что такое Genode?

   Genode OS Framework — это набор инструментов для построения специализированных операционных систем. Он предоставляет модель, в которой каждый компонент исполняется в изолированном окружении и взаимодействует с другими исключительно через явно заданные интерфейсы. Все это выглядит, как клиент-серверная архитектура, при которой каждый компонент может быть одновременно как клиентом, так и сервером, а взаимодействие реализовано засчет RPC **(Кривоватая формулировка)**. Фреймворк поддерживает несколько ядер в качестве основы (в том числе NOVA, формально верифицированное **(требует поясненй)** seL4, Fiasco.OC, linux), предоставляет готовый набор драйверов, файловых систем и сервисов, что позволяет сосредоточиться на прикладной логике, не изобретая базовую инфраструктуру заново. (https://genode.org/documentation/genode-foundations/23.05/index.html)

2. Пару слов о capability-based безопасности и recursive system structure
   Доступ к ресурсам в Genode организован в соответствии с принципами capability-based безопасности. Capability — это токен, дающий право на выполнение конкретной операции над конкретным объектом. Компонент может воспользоваться ресурсом только при наличии соответствующей capability, которую ему явно передал родительский компонент. При этом родительский компонент полностью ответственен за предоставление соответствующих capabilities дочернему и за передачу его запросов далее по иерархии. Это исключает возможность несанкционированного доступа к ресурсам в обход (иерархии), что значительно отличается от подхода привычных нам сегодня ОС, в которых процесс с достаточными привилегиями может обратиться к произвольному ресурсу. (https://genode.org/documentation/genode-foundations/23.05/index.html,    глава «Access control»)

3. Как работает Phantom на Genode?
   В рамках проекта phantomuserland-snapper **(Ссылка)** PhantomOS портируется на Genode в виде набора компонентов. Виртуальная машина Phantom (PVM) исполняется как пользовательский процесс Genode.  Для создания снапшотов, управления ими и восстановления состояния из был разработан snapper **(Ссылка)**, использующий ФС для их хранения. Такой подход позволяет Phantom пользоваться проверенными драйверами и сервисами Genode, не реализуя их самостоятельно, изоляция компонентов Genode обеспечивает дополнительный рубеж защиты, а использование микроядер позволяет добиться скромных, по сегодняшним меркам, размеров TBC, что также положительно сказывается на безопасности ОС.

### Problem statement

*введение (актуальность темы, цель, задачи, объект исследования, новизна темы исследования)*

1. Почему важно защитить снимки в покое?Как защищены снимки? (никак — актуальность) В чем заключается новизна (кто раньше этим занимался)?

   Однако некоторые проблемы безопасности все еще предстоит решить. Так, в отличие от большинства современных ОС, которые позволяют выполнять шифрование дискового пространства, PhantomOS не поддерживает это ни в каком виде. Поиск и анализ исследований также показал, что работы с PhantomOS в этом направлении не велись. При том информация, хранящаяся в снапшоте может быть более чувствительной, нежели файлы на диске, поскольку в момент снимка в памяти могут оказаться конфиденциальные данные. Это во многом схоже с проблемой безопасности снапшоов виртуальных машин **(Ссылка)**. При определенных условиях это может позволить полностью воспрои состояние и дальнейшую работу машины на устройстве злоумышленника. Из этого вытекает необходимость реализации защиты снапшотов в покое.

2. Какова цель работы?
   Таким образом, целью работы является обеспечение конфиденциальности снапшотов PhantomOS в покое путём.

3. Какие задачи работы?

   Из этого вытекают следующие цели:

   - Проанализировать сценарии использования компьютера и выделить среди них тебезопасные сценарии, которые будут поддерживаться нами
   - Написать или адаптировать компонент Genode, реализующий такой сценарий
   - Верифицировать корректность защиты: продемонстрировать недоступность содержимого снапшота
   - Оценить влияние шифрования на производительность путём сравнительного анализа скорости работы с включённым и      отключённым шифрованием

4. Что является объектом исследования?
   Объект исследования — безопасность снапшотов PhantomOS в покое в контексте порта на фреймворк Genode. Предмет исследования — механизм блочного шифрования снапшотов на основе библиотеки Tresor в среде Genode.



## Literature review

###  Вступление

- **ru**

  ```
  Эта глава посвящается обзору существующих работ, связанных с нашей темой. Глава разделена на 4 основные части, в каждой из которых будут рассмотрены труды, касающиеся соответственно
  - Section 1: Концепции ортогональной персистентности и вариантам ее реализации;
  - Section 2: PhantomOS и реализации концепции ортогональной персистентности в ней;
  - Section 3: Деталей и особенностей порта PhantomOS на Genode;
  - Section 4: Способам защиты данных в покое и их особенностям;
  ```

- **en**

  ```
  This chapter provides an overview of existing research related to our topic. The chapter is divided into four main sections, each of which will examine works pertaining to the following
  \begin{enumerate}[label=\textbf{Section \arabic*:}, leftmargin=*]
    \item The concept of orthogonal persistence and its implementation options;
    \item PhantomOS and its implementation of the concept of orthogonal persistence;
    \item Details and features of the PhantomOS port to Genode;
    \item Methods of protecting data at rest and their characteristics;
  \end{enumerate}
  ```

### Persistence

- **ru**

  ```
  Персистентность данных - это период времени, в течение которого данные существуют и используются \cite{PLA}. Именно так определяют это понятие одни из первых исследователей направления - Моррисон Р. и Аткинсон М. П.. В своей работе они выделяют категории персистентности, разделяемые на 2 группы: 
  1. Обеспечиваемые языком программирования;
  2. Обеспечиваемые средой;
  Исследователи стрематся к системам, в которых использование данных не зависит от их группы и кеатегории. Фундаментом понятия стали пинципы, сформуларованные иследователями:
  - Независимость
  - Ортогональность типа данных
  - Идентификация персистентности
  Система, следующая всем трем принципам, является ортогонально персистентной.
  Желание создания такой системы обусловлено наличием ряда приемуществ по отношению к привычным системам. К ним относятся \cite{Revisited}:
  - Повышение производительности программирования засчет упрощения семантики;
  - Избегание несистематических решений для преобразования данных и долгосрочного хранения данных;
  - Обеспечение механизмов защиты всей окружающей среды;
  - Поддержка постепенной эволюции;
  - automatically preserving referential integrity over the entire computational environment for the whole life-time of an application
  Все они в той или иной степени являются следствием упрощения модели работы с памятью для прикладного программиста. Таким образом уйдет огромный пласт кода, призванный решить вопросы ввода-вывода и сохранения данных. К ним относятся также сериализация/десериализация или инициализация/деинициализация.
  Все вышеизложенные достоинства имеют свою цену. Помимо общих трудностей, касающихся создания и внедрения новой, а уж тем более - построенной на совершенно иных, отличающихся от привычных, принципах - системы, есть и специфичные для ортогонально персистентных систем. К ним относятся:
  1. Необходимость создания стабильного объектного хранилища: В значительной степени зависит от конкретного способа исполнения, поэтому к нему мы верномся в Секции 2.2.
  2. Снижение эффективности некоторых приложений: В связи с тем, что пользователь программист не имеет полного контроля над местом нахождения данных, скорость доступа к ним может быть снижена. Аналогичная проблема присутствует при использовании виртуальной памяти.
  3. The cost of providing language independent binding mechanisms.
  Забегая вперед, скажу, что в Phantom есть решения для минимизации каждой из этих проблем, поэтому к ним мы еще вернемся в Секции 2.2. А сейчас заметим, что п. 1 в значительной степени зависит от реализации, и обобщить информацию по нему затруднительно. Пороблемы, аналогичные п. 2, касаются (пусть зачастую и в меньшей степени) и систем с виртуальной памятью, поскольку программист не имеет полного контроля над расположением данных, из-за чего скорость доступа к ним может снижаться.
  ```

- **en**

  ```
  \section{Persistence}
  
  \subsection{Persistence definitions}
  
  Data persistence refers to the period of time during which data exists and is used \cite{PLA}. This is precisely how some of the earliest researchers in the field—Morrison R. and Atkinson M. P.—define this concept. In their work, they identify categories of persistence, divided into two groups: 
  \begin{enumerate}[label=\arabic*.]
    \item Provided by the programming language;
    \item Provided by the environment;
  \end{enumerate}
  Researchers are striving to develop systems in which data usage is independent of the group or category to which the data belongs. 
  
  
  \subsection{Orthogonal persistence principles}
  
  The concept of orthogonal persistence is based on principles formulated by researchers:
  \begin{enumerate}
      \item Independence;
      \item Orthogonality of data types;
      \item Persistence Identification;
  \end{enumerate}
  In this case, independence means that data persistence does not depend on how the data is manipulated (the user cannot move data between long-term and short-term storage). The orthogonality of data types ensures that there are no cases in which data of any type cannot be persisted. The third principle means that perrsistent object identification does not relate to the type of such an jbject. A system that follows all three principles is orthogonally persistent.
  
  \subsection{Advantages of a persistent system}
  
  The desire to create such a system stems from the fact that it offers a number of advantages over conventional systems. These include \cite{Revisited}:
  \begin{enumerate}
      \item Increased programming productivity through simplified semantics;
      \item Avoiding ad hoc solutions for data transformation and long-term data storage;
      \item Providing security mechanisms for the entire environment;
      \item Support for gradual evolution;
      \item Automatically preserving referential integrity across the entire computational environment for the entire lifetime of an application
  \end{enumerate}
  All of these are, to some extent, the result of simplifying the memory management model for application programmers. This eliminates a huge amount of code that would otherwise be required to handle I/O and data storage. This also includes serialization/deserialization and initialization/deinitialization.
  
  
  \subsection{The cost of persistence}
  
  All of the advantages outlined above come at a cost. In addition to the general challenges associated with creating and implementing a new system—especially one built on principles that are entirely different from those we are accustomed to—there are also challenges specific to orthogonally persistent systems. These include:
  \begin{enumerate}[label=\arabic*.]
      \item The need for a stable object store: This depends largely on the specific implementation method, so we will return to this topic in Section 2.2.
      \item Reduced performance of some applications: Because the programmer does not have full control over the location of the data, access speed may be reduced. A similar problem exists when using virtual memory.
      \item The cost of providing language-independent binding mechanisms.
  \end{enumerate}
  Looking ahead, I will say that Phantom has solutions to minimize each of these problems, so we will return to them in Section 2.2. For now, note that point 1 depends largely on the implementation, and it is difficult to generalize information regarding it. Problems similar to point 2 also affect (albeit often to a lesser extent) systems with virtual memory, since the programmer does not have full control over the layout of the data, which can slow down access to it.
  ```

1. Что такое персистентность?
   Персистентность данных - это период времени, в течение которого данные существуют и используются \cite{PLA}. Именно так определяют это понятие одни из первых исследователей ортогональной персистентности - Моррисон Р. и Аткинсон М. П..

2. Какие виды персистентности выделяют?
   В своей работе они выделяют категории персистентности:

   - transient results in expression evaluation
   - local variables in procedure activations
   - own variables, global variables and heap items whose extent is different from their scope
   - data that exists between executions of a program
   - data that exists between various versions of a program
   - data that outlives the program.

   разделяемые на 2 группы: 

   1. Обеспечиваемые языком программирования;
   2. Обеспечиваемые средой;

   Исследователи стрематся к системам, в которых использование данных не зависит от их группы и кеатегории. Исходя из этого выделяются 3 принципа ортогональной персистентности:

   - Независимость
     Персистентность данных не зависист от способа манипуляции с ними (пользовател не может перемещать данные между долговременными и кратковременными хранилищами)
   - Ортогональность типа данных
     Не существует случаев, при которых данные какого-либо типы не могут быть персистентными
   - Идентификация персистентности
     Выбор способа идентификации и предоставления персистентных объектов  ортогонален сфере применения системы. То есть механизм идентификации персистентных объектов не связан с системой типов.

   Система, следующая всем трем принципам, является ортогонально персистентной.

3. В чем ее (ортогональной персистентности) приемущество (детально)?
   К ее достоинствам относят \cite{Revisited}:

   - Повышение производительности программирования засчет упрощения семантики;
   - Избегание несистематических решений для преобразования данных и долгосрочного хранения данных;
   - Обеспечение механизмов защиты всей окружающей среды;
   - Поддержка постепенной эволюции;
   - Automatically preserving referential integrity over the entire computational environment for the whole life-time of an application

4. В чем заключаются сложности реализации?

   Однако за все приходится платить. В случае с ортогональной персистентностью платой будет:

   1. Необходимость создания стабильного объектного хранилища: В значительной степени зависит от конкретного способа исполнения, поэтому к нему мы верномся в Секции 2.2.
   2. Снижение эффективности некоторых приложений: В связи с тем, что пользователь программист не имеет полного контроля над местом нахождения данных, скорость доступа к ним может быть снижена. Аналогичная проблема присутствует при использовании виртуальной памяти.
   3. The cost of providing language independent binding mechanisms.

   Забегая вперед, скажу, что в Phantom есть решения для решения или минимизации каждой из этих проблем, поэтому к ним мы еще вернемся в Секции 2.2. А сейчас заметим, что п. 1 в значительной степени зависит от реализации, и обобщить информацию по нему затруднительно. Пороблемы, аналогичные п. 2, касаются (пусть зачастую и в меньшей степени) и систем с виртуальной памятью, поскольку программист не имеет полного контроля над расположением данных, из-за чего скорость доступа к ним может снижаться.

5. Примеры реализации: ОС, некое окружение (возможно), прикладное ПО (что-то было для сохранения состояния процесса)

### PhantomOS

- **ru**

  ```
  PhantomOS - это операционная система с открытым исходным кодом. Автором идеи является Дмитрий Завалишин. Он же внес основной вклад в разработку системы. Как и другие современные операционные системы, Phantom имеет многозадачность\cite{Habr_MT, Habr_Preempt, Habr_Sched}, оконную подсистему \cite{Habr_UI}, сетевой стек и другие привычные атрибуты. Однако для нашей работы наибольший интерес представляет другая её особенность. PhantomOS предоставляет ортогонально персистентное окружение прикладным программам. внутри Phantom Virtual Machine (PVM), внутри которой исполняется код на языке программирования Phantom. Однако стоит отметить, что имеется и подсистема совместимости POSIX. Доступ по произвольному адресу исключён. В Phantom уже реализованы такое подсистемы, как:
  - Kernel itself: threads, synchronization, persistent memory management;
  - Bytecode virtual machine - running native applications;
  - Posix layer - runs Linux compatible (but not yet persistent) code;
  - Graphics subsystem - Windows, controls, UI;
  - Networking (TCP/IP);
  - Phantom language compiler - the most native userland language;
  - Java to Phantom translator - work in progress;
  - Python to Phantom translator - just started;
  Все это указано в документации \cite{Phantom_docs}.
  
  Дмитрий Завалишин считает, что сохранить состояние всей машины невозможно, но для достижения персистентности это и не требуется \cite{Habr_Persistent_Mem}. Поэтому далее будем говорить о механизмах обеспечения персистентной среды. Персистентность в PhantomOS достигается постраничным отображением всей памяти (PVM) на диск и периодическим ее сохранением. Запуском процесса моментального снимка руководит ядро. Оно делает снапшоты, предварительно загнав программы в такое состояние, при котором оно полностью представлено в памяти. То есть для этого переносится и вся информация из регистров процессора. Только при этой кратковременной подготовке происходит блокировка прикладных программ. Диск при этом переводится в read-only режим, но это не приводит к блокировке на протяжении всего процесса сброса снапшота на диск благодаря использованию CoW: если страница была записана в снапшот, доступу к ней ничего не препятствует; в противном случае создается копия, которой и оперирует программа, а изначальная страница перемещается ближе к началу очереди на сохранение \cite{Habr_Persistent_Mem}. Запись всей памяти на диск при каждом снимке - весьма дорогостоящая операция, в которой нет необходимости. Для оптимизации этого процесса на диск записывается только инкремент, а также задействуется превентивная запись. Последнее, кстати, используется для двух целей:
  - Для ускорения процесса записи снапшота;
  - Для удовлетворения повышенного спроса на память во время снимка;
  ```

- **en**

  ```
  \section{PhantomOS}
  
  \subsection{PhantomOS Overview}
  
  PhantomOS is an open-source operating system. The concept was conceived by Dmitry Zavalishin, who also made the primary contribution to the system’s development. Like other modern operating systems, Phantom features multitasking \cite{Habr_MT, Habr_Preempt, Habr_Sched}, a windowing subsystem \cite{Habr_UI}, a network stack, and other familiar attributes. However, for our work, another feature is of greatest interest. PhantomOS provides an orthogonally persistent environment for applications within the Phantom Virtual Machine (PVM), where code is executed in the Phantom programming language. However, it is worth noting that there is also a POSIX compatibility subsystem. Phantom already implements subsystems such as:
  \begin{itemize}
      \item Kernel itself: threads, synchronization, persistent memory management;
      \item Bytecode virtual machine - running native applications;
      \item POSIX layer - runs Linux-compatible (but not yet persistent) code;
      \item Graphics subsystem - Windows, controls, UI;
      \item Networking (TCP/IP);
      \item Phantom language compiler - the most native userland language;
      \item Java to Phantom translator - work in progress;
      \item Python to Phantom translator - just started;
  \end{itemize}
  All of this is specified in the documentation \cite{Phantom_docs}.
  
  Dmitry Zavalishin believes that it is impossible to preserve the state of the entire machine, but this is not necessary to achieve persistence \cite{Habr_Persistent_Mem}. Therefore, we will now discuss the mechanisms for ensuring a persistent environment. Persistence in PhantomOS is achieved by page-by-page mapping of the entire memory (PVM) to disk and periodically saving it. The kernel manages the snapshot process. It takes snapshots after first bringing the programs to a state where they are fully represented in memory. That is, all information from the processor registers is also transferred for this purpose. Application programs are blocked only during this brief preparation phase. The disk is switched to read-only mode during this process, but this does not cause a lock throughout the entire process of writing the snapshot to disk thanks to the use of CoW: if a page has been written to the snapshot, nothing prevents access to it; otherwise, a copy is created, which the program operates on, and the original page is moved closer to the front of the save queue \cite{Habr_Persistent_Mem}. Writing the entire memory to disk with every snapshot is a very costly operation that is unnecessary. To optimize this process, only the increment is written to disk, and preemptive writing is also used. The latter, by the way, serves two purposes:
  \begin{itemize}
      \item To speed up the snapshot writing process;
      \item To meet the increased demand for memory during the snapshot;
  \end{itemize}
  
  One of the goals behind the system’s design is to make life easier for programmers. They no longer need to manage memory; instead, the garbage collector handles this task. Other key features of the system include a global address space and the requirement that objects can only be accessed via a reference. Access via arbitrary addresses is prevented, which has a positive effect on security. It is in this environment that the garbage collector operates. The large amount of virtual memory and the inability to use stop-world algorithms led to a strategy of using two garbage collectors \cite{Habr_GC1}:
  \begin{enumerate}
      \item Partial;
      \item Full;
  \end{enumerate}
  The first is fast, inexpensive, and possibly incomplete. It is assumed that it cleans up garbage in RAM without waiting for objects to be flushed to disk. It runs continuously and is currently implemented based on counting the number of references to an object. The second should be full, but it can run periodically thanks to the presence of the first. At the same time, a stop-world implementation is also possible due to the persistence of garbage: that is, what was garbage in the old snapshot will remain garbage in the new one. Based on this idea, it is proposed to perform garbage collection on the old snapshot, although not the entire snapshot may be used for this, but only a reflection of its memory state. However, there are still a number of problems that need to be solved before its implementation \cite{Habr_GC2}.
  ```

1. В чем заключается философия системы? Что про нее можно сказать (какие особенности можно выделить)?
   PhantomOS - это операционная система с открытым исходным кодом. Автором идеи является Дмитрий Завалишин. Он же внес основной вклад в разработку системы. Как и другие современные операционные системы, Phantom имеет многозадачность\cite{Habr_MT, Habr_Preempt, Habr_Sched}, оконную подсистему \cite{Habr_UI}, сетевой стек и другие привычные атрибуты. Однако для нашей работы наибольший интерес представляет другая её особенность. PhantomOS предоставляет ортогонально персистентное окружение прикладным программам. внутри Phantom Virtual Machine (PVM), внутри которой исполняется код на языке программирования Phantom. Однако стоит отметить, что имеется и подсистема совместимости POSIX. Доступ по произвольному адресу исключён. В Phantom уже реализованы такое подсистемы, как:

   - Kernel itself: threads, synchronization, persistent memory management;
   - Bytecode virtual machine - running native applications;
   - Posix layer - runs Linux compatible (but not yet persistent) code;
   - Graphics subsystem - Windows, controls, UI;
   - Networking (TCP/IP);
   - Phantom language compiler - the most native userland language;
   - Java to Phantom translator - work in progress;
   - Python to Phantom translator - just started;

   Все это указано в документации \cite{Phantom_docs}.

2. Как работает процесс создания снимка на оригинальной версии ОС?
   Дмитрий Завалишин считает, что сохранить состояние всей машины невозможно, но для достижения персистентности это и не требуется \cite{Habr_Persistent_Mem}. Поэтому далее будем говорить о механизмах обеспечения персистентной среды. Персистентность в PhantomOS достигается постраничным отображением всей памяти (PVM) на диск и периодическим ее сохранением. Запуском процесса моментального снимка руководит ядро. Оно делает снапшоты, предварительно загнав программы в такое состояние, при котором оно полностью представлено в памяти. То есть для этого переносится и вся информация из регистров процессора. Только при этой кратковременной подготовке происходит блокировка прикладных программ. Диск при этом переводится в read-only режим, но это не приводит к блокировке на протяжении всего процесса сброса снапшота на диск благодаря использованию CoW: если страница была записана в снапшот, доступу к ней ничего не препятствует; в противном случае создается копия, которой и оперирует программа, а изначальная страница перемещается ближе к началу очереди на сохранение \cite{Habr_Persistent_Mem}. Запись всей памяти на диск при каждом снимке - весьма дорогостоящая операция, в которой нет необходимости. Для оптимизации этого процесса на диск записывается только инкремент, а также задействуется превентивная запись. Последнее, кстати, используется для двух целей:

   - Для ускорения процесса записи снапшота;
   - Для удовлетворения повышенного спроса на память во время снимка;

3. Особенности системы и среды
   Один из замыслов создания системы - облегчение жизни программиста. У него отпадает необходимость следить за памятью, а вместо него эту работу выполняет сборщик мусора. К особенностям системы также относится глобальное адресное пространство и возможность взаимодействия с объектами исключительно при наличии ссылки на него. Возмодность доступа по произвольному адресу исключается, что положителььно сказывается на безопасности. Именно в такой среде и работает сборщику мусора. Большой объем виртуальной памяти и невозможность использования stop-world алгоритмов привели к стратегии использования двух сборщиков мусора \cite{Habr_GC1}:

   1. Неполный;
   2. Полный;

   Первый - быстрый, недорогой, возможно — неполный. Предполагается, что он производит очистку мусора в оперативной памяти, не дожидаясь сброса объектов на диск. Он работает постоянно, на данный момент реализован на принципе подсчёта числа ссылок на объект. Второй должен быть полным, но его запуск может быть периодичным, благодаря наличию первого. При этом допускается и stop-world реализация благодаря постоянству мусора: то есть то, что было мусором в старом снимке, останется мусором и в новом. Основываясь на этой идее, предполагается проводить сборку мусора на старом снимке, хотя для этого может использоваться не весь снапшот, а лишь отражение его состояния памяти. Однако все еще существует ряд проблем, которые предстоит решить перед его реализацией \cite{Habr_GC2}.

4. Какова стоимость персистентности?

### PhantomOS на Genode

- **ru**

  ```
  Как уже было сказано, сейчас PhantomOS портируется на Genode фреймворк. Делается это для улучшения \cite{Antonov_Antonov};
  
  - Надежности;
  - Безопасности;
  - Interoperability;
  
  Таким образом планируется достичь требуемого качества. Одну из главных проблем надежности - новое ядро со встроенными драйверами- планируется решить путем использования ядер, поддерживаемых Genode, среди которых множество микроядер, отличающихся повышенной надежностью. Также предлагается заменить нове, недостаточно протестированне компоненты уже проверенными с тем же функционалом. Это также решит вопрос Interoperability, так как не нужно будет писать уже существующие компонент, и можо будет сосредоточиться на логике системы.  К вопросу безопасности разработчики Genode подошли основательно. Фреймворк использует capability-based систему безопасности. Зачастую в качестве ядра системы используются микроядра. Благодаря этому удается в значительной степени изолировать компоненты и более гибко настроить доступ к резурсам, что позволяет достичь минимальной TCB. Таким образом достигается минимальная поверхность атаки, что крайне благоприятно сказывается на надежности.
  
  Изначально была проведена работа по портированию системы \cite{Antonov_Anton}. По большому счету, задача заключается в переносе перистентной среды на фреймворк. Помимо PVM были проведены работы с HAL, libc, механизмом выделения физической памяти. Некоторые функции были заменены заглушками. Результатом работы стал работающий проект isomem \cite{GitHub_isomem}. В нем также присутствует видеозаглушка в виде подсистемы окон с ограниченным функционалом. Он конфигурируется запускается как отдельный самостоятельный компонент Genode. После этого были работы по разработке персистентной сетевой подсистемы для порта \cite{Brisilin_Anton} и  внедрению среды выполнения байт-кода WASM \cite{Samburskiy_Kirill}. Также менялся механизм создания снапшотов, результатом чего стало появление проекта snapper \cite{GitHub_Snapper}.
  
  Snapper в текущей версии (Snapper 2.0) призван решить проблемы, с которыми столкнулась реализация предыдущего механизма - "superblock" \cite{FOSDEM_Snapper, FOSDEM_Snapper_paper}. Он повышает эффективность использования дискового пространства при хранении нескольких снапшотов. Ранее снапшоты хранились как моголитные объекты в памяти, что приводило к большому количеству копий данных при хранении нескольких снапшотов. Для неизененных страниц Snapper добавляет ссылку в новом поколении, не копируя все данные снова. Возможность восстановления после сбоев обеспечивается использованием файловых систем, для которых уже существуют такие инструменты, например: ext4 и fsck. Использование файловой системы для хранения снапшотов может быть более удобно в процессе разработки и отладки. Был добавлен контроль целостности.  Помимо этого система стала конфигурируемой в контексте повышения надежности благодаря введению такого параметра, как redundancy. При привышении заданного числа ссылок на файл создается его копия, и все поколения, использующие этот файл, линкуются как с изначальным файлом, так и с его копией. Это позволяет опытному администратору самостоятельно настроить баланс надежности и использования хранилища.
  ```

- **en**

  ```
  As previously mentioned, PhantomOS is currently being ported to the Genode framework. This is being done to improve \cite{Antonov_Antonov};
  \begin{itemize}
      \item Reliability;
      \item Security;
      \item Interoperability;
  \end{itemize}
  - 
  In this way, we plan to achieve the required quality. One of the main reliability issues—the new kernel with built-in drivers—is planned to be resolved by using kernels supported by Genode, including many microkernels known for their high reliability. It is also proposed to replace new, insufficiently tested components with already proven ones offering the same functionality. This will also resolve the issue of interoperability, as there will be no need to rewrite existing components, and the focus can shift to the system’s logic.  The Genode developers have taken a thorough approach to security. The framework uses a capability-based security system. Microkernels are often used as the system kernel. This makes it possible to isolate components to a significant degree and configure access to resources more flexibly, allowing for a minimal TCB. This results in a minimal attack surface, which has an extremely positive effect on reliability.
  
  Initially, work was carried out to port the \cite{Antonov_Anton} system. Essentially, the task involves migrating the peristent environment to the framework. In addition to PVM, work was done on HAL, libc, and the physical memory allocation mechanism. Some functions were replaced with placeholders. The result of this work was a working isomem project \cite{GitHub_isomem}. It also includes a video placeholder in the form of a window subsystem with limited functionality. It is configured and runs as a separate, standalone Genode component. Subsequently, work was carried out on developing a persistent network subsystem for the \cite{Brisilin_Anton} port and implementing the WASM bytecode runtime \cite{Samburskiy_Kirill}. The snapshot creation mechanism was also modified, resulting in the creation of the snapper project \cite{GitHub_Snapper}.
  
  The current version of Snapper (Snapper 2.0) is designed to address the issues encountered in the implementation of the previous mechanism—the “superblock” \cite{FOSDEM_Snapper, FOSDEM_Snapper_paper}. It improves disk space efficiency when storing multiple snapshots. Previously, snapshots were stored as monolithic objects in memory, which resulted in a large number of data copies when storing multiple snapshots. For unchanged pages, Snapper adds a link in the new generation without copying all the data again. The ability to recover from failures is ensured by using file systems for which such tools already exist, for example: ext4 and fsck. Using a file system to store snapshots can be more convenient during development and debugging. Integrity checks have been added.  In addition, the system has become configurable in terms of reliability through the introduction of a parameter called redundancy. If the specified number of links to a file is exceeded, a copy of the file is created, and all generations using this file are linked to both the original file and its copy. This allows an experienced administrator to independently configure the balance between reliability and storage usage.
  ```

**Причина портирования**

Как уже было сказано, сейчас PhantomOS портируется на Genode фреймворк. Делается это для улучшения \cite{Antonov_Antonov};

- Надежности;
- Безопасности;
- Interoperability;

Таким образом планируется достичь требуемого качества. Одну из главных проблем надежности - новое ядро со встроенными драйверами- планируется решить путем использования ядер, поддерживаемых Genode, среди которых множество микроядер, отличающихся повышенной надежностью. Также предлагается заменить нове, недостаточно протестированне компоненты уже проверенными с тем же функционалом. Это также решит вопрос Interoperability, так как не нужно будет писать уже существующие компонент, и можо будет сосредоточиться на логике системы.  К вопросу безопасности разработчики Genode подошли основательно. Фреймворк использует capability-based систему безопасности. Зачастую в качестве ядра системы используются микроядра. Благодаря этому удается в значительной степени изолировать компоненты и более гибко настроить доступ к резурсам, что позволяет достичь минимальной TCB. Таким образом достигается минимальная поверхность атаки, что крайне благоприятно сказывается на надежности.

**Затронутые компоненты**

Изначально была проведена работа по портированию системы \cite{Antonov_Anton}. По большому счету, задача заключается в переносе перистентной среды на фреймворк. Помимо PVM были проведены работы с HAL, libc, механизмом выделения физической памяти. Некоторые функции были заменены заглушками. Результатом работы стал работающий проект isomem \cite{GitHub_isomem}. В нем также присутствует видеозаглушка в виде подсистемы окон с ограниченным функционалом. Он конфигурируется запускается как отдельный самостоятельный компонент Genode. После этого были работы по разработке персистентной сетевой подсистемы для порта \cite{Brisilin_Anton} и  внедрению среды выполнения байт-кода WASM \cite{Samburskiy_Kirill}. Также менялся механизм создания снапшотов, результатом чего стало появление проекта snapper \cite{GitHub_Snapper}.

**Snapper**

Snapper в текущей версии (Snapper 2.0) призван решить проблемы, с которыми столкнулась реализация предыдущего механизма - "superblock" \cite{FOSDEM_Snapper, FOSDEM_Snapper_paper}. Он повышает эффективность использования дискового пространства при хранении нескольких снапшотов. Ранее снапшоты хранились как моголитные объекты в памяти, что приводило к большому количеству копий данных при хранении нескольких снапшотов. Для неизененных страниц Snapper добавляет ссылку в новом поколении, не копируя все данные снова. Возможность восстановления после сбоев обеспечивается использованием файловых систем, для которых уже существуют такие инструменты, например: ext4 и fsck. Использование файловой системы для хранения снапшотов может быть более удобно в процессе разработки и отладки. Был добавлен контроль целостности.  Помимо этого система стала конфигурируемой в контексте повышения надежности благодаря введению такого параметра, как redundancy. При привышении заданного числа ссылок на файл создается его копия, и все поколения, использующие этот файл, линкуются как с изначальным файлом, так и с его копией. Это позволяет опытному администратору самостоятельно настроить баланс надежности и использования хранилища.



1. В чем отличие Phantom на Genode
   1. Какие компоненты и как поменялись?
   2. Какие genode компоненты использует Phantom
2. Что такое Snapper?
3. Как он работает?
4. Какие компоненты использует?
5. В каком виде и где хранит данные?

### Защита данных в покое

- **ru**

  ```
  Snapper внес вклад в улучшение безопасности снапшотов, добавив контрольь целостности, однако ни он, ни какой-либо другой компонент не позволяют достичь безопасного хранения. Они лежат на диске в plaintext, что может стать серьезной проблемой безопасности при многих сценариях использования устройства.
  
  Данными в покое называют любые данные, хранящиеся на персистентных носителях и не участвующие в активной передаче или обработке. Они статичны и подвержены угрозам связанным с физическим или логическим доступом. На диске могут располагаться чувствительные данные, которые могут быть считаны или перезаписаны. В контексте PhantomOS ситуация осложняется тем, что снапшот делается независимо от того, какие данные находятся в памяти, и в него могут попасть секретные данные, даже если на прикладном уровне программы предусмотрена их защита собственными средствами \cite{SNIA_PM_Security}. Это схоже с проблемой защиты снапшотов виртуальных машин \cite{VM_Security}.
  
  Способы защиты данных должны соответствовать модели угроз, которая, в свою очередь, напрямую зависит от сценариев использования устройства. В этом подразделе будет проведен лишь краткий обзор таких способов без подробного объяснения выбора того или иного способа.
  
  В данном случае мы говорим о шифровании хранилища. Мы не рассматриваем варианты с надежными физическими и/или логическими ограничениями доступа. Шифрование можно классифицировать по полноте \cite{NIST_SP800111} :
  
  - Полное шифрование диска;
  - Шифрование томов и виртуальных дисков;
  - Шифрование файлов и папок;
  
  и по уровню  \cite{CryptoFS}:
  
  - Блочный уровень;
  - Уровень файловой системы (обобщено, делится на несколько пунктов ко количеству и удалению ФС);
  - Уровень прикладных приложений;
  
  Это уровни используемые на уровне ОС и выше. Каждый более высокий уровень оставляет метаданные более низкого уровня открытыми. Помимо них возможны реализация на аппаратном низком уровне, что будет прозрачно для ОС. Ввиду отсутствия в Genode проверенных таких систем такого или инструментов для их создания, этот уровень рассматриваться не будет.
  
  В данной работае мы будем заниматься внедрением среды для безопасного хранения снапшотов. Это будет хранилище с шщифрованием на блосном уровне, что повысит безопасность в сравнении с шифрованием на уровне файловой системы и выше, скрыв метаинформацию и усложнив процесс получения данных о системе. Этому также способствует то, что в Genode уже существует библиотека \cite{GitHub_tresor} для этого и тестовый компонент \cite{GitHub_file_vault, Page_file_vault}.
  ```

- **en**

  ```
  Snapper has helped improve snapshot security by adding integrity checks, but neither it nor any other component ensures secure storage. They are stored on disk in plaintext, which can pose a serious security risk in many device usage scenarios.
  
  Data at rest refers to any data stored on persistent storage media that is not currently being actively transmitted or processed. It is static and vulnerable to threats related to physical or logical access. Sensitive data may be stored on the disk and could be read or overwritten. In the context of PhantomOS, the situation is complicated by the fact that a snapshot is taken regardless of what data is in memory, and secret data may end up in it, even if the application level of the program provides for its protection using its own means \cite{SNIA_PM_Security}. This is similar to the problem of protecting virtual machine snapshots \cite{VM_Security}.
  
  Data protection methods must align with the threat model, which, in turn, depends directly on the device’s usage scenarios. This subsection provides only a brief overview of such methods, without a detailed explanation of why a particular method might be chosen.
  In this case, we are discussing storage encryption. We do not consider options involving robust physical and/or logical access controls. Encryption can be classified by scope \cite{NIST_SP800111}:
  - Full disk encryption;
  - Volume and virtual disk encryption;
  - File and folder encryption;
  and by level \cite{CryptoFS}:
  - Block level;
  - File system level (generalized, divided into several points regarding the number and removal of file systems);
  - Application level;
  These are the levels used at the OS level and above. Each higher level leaves the metadata of the lower level exposed. In addition to these, implementation at a low-level hardware level is possible, which would be transparent to the OS. Due to the absence in Genode of verified systems of this type or tools for their creation, this level will not be considered.
  
  In this paper, we will focus on implementing an environment for the secure storage of snapshots. This will be a block-level encrypted storage system, which will enhance security compared to file-system-level encryption and higher, by hiding metadata and making it more difficult to extract system information. This is also facilitated by the fact that Genode already has a library \cite{GitHub_tresor} for this purpose and a test component \cite{GitHub_file_vault, Page_file_vault}.
  ```

Snapper внес вклад в улучшение безопасности снапшотов, добавив контрольь целостности, однако ни он, ни какой-либо другой компонент не позволяют достичь безопасного хранения. Они лежат на диске в plaintext, что может стать серьезной проблемой безопасности при многих сценариях использования устройства.

1. Зачем нужна защита данных в покое?
2. В чем отличие данных снапшотов и данных на диске?
3. Какие есть способы защиты данных в покое?
   1. Какие уровни шифрования бывают?
   2. В чем приемущества и недостатки шифрования на разных уровнях?
   3. Чем мы жертвуем? как это сказывается на производительности? (Аппаратное смягчение последствий)
   4. Есть ли другие способы защиты данных в покое?
4. Какой подход обычно применяется (или несколько)?
5. Какой способ предпочтителен для нас и почему? (возможно, это уже другая глава)





\subsection{Зачем защищать снапшоты в покое}

Данными в покое называют любые данные, хранящиеся на персистентных носителях и не участвующие в активной передаче или обработке. Они статичны и подвержены угрозам связанным с физическим или логическим доступом. На диске могут располагаться чувствительные данные, которые могут быть считаны или перезаписаны. В контексте PhantomOS ситуация осложняется тем, что снапшот делается независимо от того, какие данные находятся в памяти, и в него могут попасть секретные данные, даже если на прикладном уровне программы предусмотрена их защита собственными средствами \cite{SNIA_PM_Security}. Это схоже с проблемой защиты снапшотов виртуальных машин \cite{VM_Security}.



\subsection{Ways to protect stored data}



Способы защиты данных должны соответствовать модели угроз, которая, в свою очередь, напрямую зависит от сценариев использования устройства. В этом подразделе будет проведен лишь краткий обзор таких способов без подробного объяснения выбора того или иного способа.

В данном случае мы говорим о шифровании хранилища. Мы не рассматриваем варианты с надежными физическими и/или логическими ограничениями доступа. Шифрование можно классифицировать по полноте \cite{NIST_SP800111} :

- Полное шифрование диска;
- Шифрование томов и виртуальных дисков;
- Шифрование файлов и папок;

и по уровню  \cite{CryptoFS}:

- Блочный уровень;
- Уровень файловой системы (обобщено, делится на несколько пунктов ко количеству и удалению ФС);
- Уровень прикладных приложений;

Это уровни используемые на уровне ОС и выше. Каждый более высокий уровень оставляет метаданные более низкого уровня открытыми. Помимо них возможны реализация на аппаратном низком уровне, что будет прозрачно для ОС. Ввиду отсутствия в Genode проверенных таких систем такого или инструментов для их создания, этот уровень рассматриваться не будет. 



\subsection{Our work}



В данной работае мы будем заниматься внедрением среды для безопасного хранения снапшотов. Это будет хранилище с шщифрованием на блосном уровне, что повысит безопасность в сравнении с шифрованием на уровне файловой системы и выше, скрыв метаинформацию и усложнив процесс получения данных о системе. Этому также способствует то, что в Genode уже существует библиотека \cite{GitHub_tresor} для этого и тестовый компонент \cite{GitHub_file_vault, Page_file_vault}.





### Methodology

- **ru**

  ```
  В этой главе рассматриваются требования к системе и план реализации. Сначала мы опишем модель использования компьютера и выделим безопасные сценарии, которые собираемся поддерживать. Затем выдвенем требования к компоненту. После этого расскажем о том, какой компонент будет адаптирован и каким образом.
  ```

- **en**

  ```
  This chapter discusses system requirements and the implementation plan. First, we will describe a model of computer usage and identify the secure scenarios we intend to support. Next, we will outline the requirements for the component. After that, we will explain which component will be adapted and how.
  ```

В этой главе рассматриваются требования к системе и план реализации. Сначала мы опишем модель использования компьютера и выделим безопасные сценарии, которые собираемся поддерживать. Затем выдвенем требования к компоненту. После этого расскажем о том, какой компонент будет адаптирован и каким образом.



\section{Use cases}

- **ru**

  ```
  Чтобы выдвинуть наиболее подходящие требования к компоненту, необходимо определиться со сценарием использования устройства. Для этого была построена схема сценариев использования компьютера, на ней были отмечены небезопасные и безопасные сценарии. Из безопасных были отобраны поддерживаемые, реализуемые в рамках данной работы.
  
  Прежде чем описывать саму схему, стоит сказать о том, что на ней изображено и как она строилась. Схема представляет из себя граф, обход котороо начинается и заканчивается состоянием "Poewer off". На ней изображен весь цикл работы компьютера. Каждый возможный путь является отдельным сценарием использования. При ее составлении мы стремились описать абсолютно все сценарии использования компьютера, сгруппировав их по важным для нас признакам. Группировка делалась без частичных пересечений или полных совпадений, фактическая работа может и, зачастую, будет описываться комбинациями таких сценариев. Состояния, от которых не зависит последующее состояние или безопасность всего пути, были исключены для простоты. Также не рассматривались априори неверные случаи, например, получение валидного фактора аутентификации от недоверенного пользователя.
  
  В первую очередь было выявлено 4 начальных состояния. Они были получены разделением по признакам способа доступа и доверенности пользователя. В данном случае удаленный доступ подразумевает возможность взаисмодействия с компьютером исключительно через доверенные устройства ввода-вывода. Произвольный доступ подразумевает отсутствие каких-либо ограничений во взаимодействии. Фактически, множество вариантов удаленного доступа является подмножеством вариантов произвольного доступа. Разумеется, при этом проблемы с безопасностью на аппаратном уровне могут привести к компрометации системы. Явные эксплойты ниже уровня ОС (аппаратные, прошивки) не рассматриваются в работе и не влияют на оценку безопасности сценария. Аналогично и с пользователями. Доверенный пользователь - это тот, доступ которого к компьютеру легетимен. По большому счету, задача сводится к тому, чтобы из всех пользователей только доверенные могли получить доступ.
  
  Сценарии только с доверенными способами упрощены максимально, так как в случае такого разделения поставленная задача всегда выполняется. Для определения доверенных пользователей используется ПО для авторицазии, которое мы считаем надежным. Также считаем, что факторы аутентификации подбираются таким образом, что его валидность однозначно определяет доверенного пользователя. Среди оставшихся вариантов значительно упрощены те, что не имеют авторизации, и удаленные.
  
  \subsection{Supporteddd use case}
  
  Для нашей работы были выбраны безопасные сценарии с авторизацией, при которых произвольный доступ к устройству имеют все пользователи. От небезопасных их отличает то, что они гарантируют зашифрованное состояние снапшота по завершении работы. Также отметим, что гами не рассматриваются сценарии, при которых после предоставления доступа доверенному ользователю, его получил недоверенный. То есть авторицацию проходит кждый пользователь без исключения. На схеме опущены детали выбора фактора аутентификации и типа шифрования. Использование внешнего хранилища для ключа обусловлено простотой реалицации при приемлемым для нас уровне безопасности. Это позволит объединить аутентификацию с дешифрованием.
  
  
  ```

- **en**

  ```
  To identify the most appropriate requirements for the component, it is necessary to define the device’s usage scenarios. To this end, a diagram of computer usage scenarios was created, with unsafe and safe scenarios marked on it. From among the safe scenarios, those that are supported and can be implemented within the scope of this project were selected.
  
  \subsection{All use cases}
  
  Before describing the diagram itself, it’s worth mentioning what it depicts and how it was constructed. The diagram is a graph whose cycle begins and ends with the “Power off” state. It illustrates the entire computer operation cycle. Each possible path represents a distinct usage scenario. When creating it, we aimed to describe absolutely all possible computer usage scenarios, grouping them according to criteria that were important to us. The grouping was done without partial overlaps or complete overlaps; actual operation may, and often will, be described by combinations of such scenarios. States on which the subsequent state or the security of the entire path does not depend were excluded for simplicity. We also did not consider a priori invalid cases, such as receiving a valid authentication factor from an untrusted user.
  
  \begin{figure}[H]
      \centering
      \includegraphics[width=\linewidth, height=\textheight, keepaspectratio]{use_cases_supported.pdf}
      \caption{Use case scheme}
      \label{fig:use_cases}
  \end{figure}
  
  Before describing the diagram itself, it’s worth mentioning what it depicts and how it was constructed. The diagram is a graph whose cycle begins and ends with the “Power off” state. It illustrates the entire computer operation cycle. Each possible path represents a distinct usage scenario. When creating it, we aimed to describe absolutely all possible computer usage scenarios, grouping them according to criteria that were important to us. The grouping was done without partial overlaps or complete overlaps; actual operation may, and often will, be described by combinations of such scenarios. States on which the subsequent state or the security of the entire path does not depend were excluded for simplicity. Cases that are a priori incorrect were also not considered, such as receiving a valid authentication factor from an untrusted user.
  
  First, four initial states were identified. These were derived by classifying the states based on access method and user privileges. In this context, remote access refers to the ability to interact with the computer exclusively through trusted input/output devices. Arbitrary access refers to the absence of any restrictions on interaction. In fact, the set of remote access options is a subset of the set of arbitrary access options. Of course, security issues at the hardware level can lead to system compromise. Exploits below the OS level (hardware, firmware) are not considered in this work and do not affect the security assessment of the scenario. The same applies to users. A trusted user is one whose access to the computer is legitimate. By and large, the task boils down to ensuring that, of all users, only trusted ones can gain access.
  
  Scenarios involving only trusted methods are simplified as much as possible, since with such a separation, the task at hand is always accomplished. To identify trusted users, we use authentication software that we consider reliable. We also believe that authentication factors are selected in such a way that their validity unambiguously identifies a trusted user. Among the remaining options, those without authentication and remote access are significantly simplified
  
  \subsection{Supported use cases}
  
  For our work, we selected secure authorization scenarios in which all users have unrestricted access to the device. These scenarios differ from insecure ones in that they guarantee the snapshot remains encrypted upon completion of the operation. We also note that scenarios in which, after granting access to a trusted user, an untrusted user gains access are not considered. In other words, every user, without exception, goes through authorization. The diagram omits details regarding the choice of authentication factor and encryption type. The use of external storage for the key is due to the simplicity of implementation while maintaining a security level acceptable to us. This will allow us to combine authentication with decryption.
  ```

Чтобы выдвинуть наиболее подходящие требования к компоненту, необходимо определиться со сценарием использования устройства. Для этого была построена схема сценариев использования компьютера, на ней были отмечены небезопасные и безопасные сценарии. Из безопасных были отобраны поддерживаемые, реализуемые в рамках данной работы.



\subsection{All use cases}

Прежде чем описывать саму схему, стоит сказать о том, что на ней изображено и как она строилась. Схема представляет из себя граф, обход котороо начинается и заканчивается состоянием "Poewer off". На ней изображен весь цикл работы компьютера. Каждый возможный путь является отдельным сценарием использования. При ее составлении мы стремились описать абсолютно все сценарии использования компьютера, сгруппировав их по важным для нас признакам. Группировка делалась без частичных пересечений или полных совпадений, фактическая работа может и, зачастую, будет описываться комбинациями таких сценариев. Состояния, от которых не зависит последующее состояние или безопасность всего пути, были исключены для простоты. Также не рассматривались априори неверные случаи, например, получение валидного фактора аутентификации от недоверенного пользователя.

В первую очередь было выявлено 4 начальных состояния. Они были получены разделением по признакам способа доступа и доверенности пользователя. В данном случае удаленный доступ подразумевает возможность взаисмодействия с компьютером исключительно через доверенные устройства ввода-вывода. Произвольный доступ подразумевает отсутствие каких-либо ограничений во взаимодействии. Фактически, множество вариантов удаленного доступа является подмножеством вариантов произвольного доступа. Разумеется, при этом проблемы с безопасностью на аппаратном уровне могут привести к компрометации системы. Явные эксплойты ниже уровня ОС (аппаратные, прошивки) не рассматриваются в работе и не влияют на оценку безопасности сценария. Аналогично и с пользователями. Доверенный пользователь - это тот, доступ которого к компьютеру легетимен. По большому счету, задача сводится к тому, чтобы из всех пользователей только доверенные могли получить доступ.

Сценарии только с доверенными способами упрощены максимально, так как в случае такого разделения поставленная задача всегда выполняется. Для определения доверенных пользователей используется ПО для авторицазии, которое мы считаем надежным. Также считаем, что факторы аутентификации подбираются таким образом, что его валидность однозначно определяет доверенного пользователя. Среди оставшихся вариантов значительно упрощены те, что не имеют авторизации, и удаленные.

\subsection{Supported use cases}



Для нашей работы были выбраны безопасные сценарии с авторизацией, при которых произвольный доступ к устройству имеют все пользователи. От небезопасных их отличает то, что они гарантируют зашифрованное состояние снапшота по завершении работы. Также отметим, что гами не рассматриваются сценарии, при которых после предоставления доступа доверенному ользователю, его получил недоверенный. То есть авторицацию проходит кждый пользователь без исключения. На схеме опущены детали выбора фактора аутентификации и типа шифрования. Использование внешнего хранилища для ключа обусловлено простотой реалицации при приемлемым для нас уровне безопасности. Это позволит объединить аутентификацию с дешифрованием.   



\section{Requirements}

- **ru**

  ```
  Сценарий диктует следующие требования к реализации:
  
  - Обязательная авторизация;
  - Хранение ключей на внешнем носителе;
  - Шифрование снапшотов;
  - Гарантия хранения только зашифрованных снапшотов;
  
  Первые два требования можно объединить. Так как для надежного шифрования требуется длинный ключ, флешка с ним станет надежным фактором, который трудно подделать. Это упростит работу без потери качества результата. Как уже было сказано ранее, в Genode ест библиотека для шифрования на блочном уровне - Tresor. Она позволит добиться выполнения последних двух требований. Шифрование будет проводиться при записи каждого блока, что гарантирует попадание на диск только зашифрованных данных.
  ```

- **en**

  ```
  The scenario dictates the following implementation requirements:
  \begin{enumerate}
      \item Mandatory authorization;
      \item Storage of keys on an external medium;
      \item Encryption of snapshots;
      \item Guarantee that only encrypted snapshots are stored;
  \end{enumerate}
  The first two requirements can be combined. Since a long key is required for reliable encryption, a USB drive containing it will serve as a secure factor that is difficult to forge. This will simplify the process without compromising the quality of the result. As mentioned earlier, Genode includes a library for block-level encryption—Tresor. It will enable the fulfillment of the last two requirements. Encryption will occur upon writing each block, ensuring that only encrypted data is written to the disk.
  
  ```

Сценарий диктует следующие требования к реализации:

- Обязательная авторизация;
- Хранение ключей на внешнем носителе;
- Шифрование снапшотов;
- Гарантия хранения только зашифрованных снапшотов;

Первые два требования можно объединить. Так как для надежного шифрования требуется длинный ключ, флешка с ним станет надежным фактором, который трудно подделать. Это упростит работу без потери качества результата. Как уже было сказано ранее, в Genode ест библиотека для шифрования на блочном уровне - Tresor. Она позволит добиться выполнения последних двух требований. Шифрование будет проводиться при записи каждого блока, что гарантирует попадание на диск только зашифрованных данных.



\section{Implementation plan}

- **ru**

  ```
  В Genode уже есть компонент file_vault, используемый для создания безопасного хранилища. Для шифрования используется библиотека Tresor. Планируется адаптироват этот компонент для шифрования снапшотов. Для этого будет убран весь нетребуемый функционал, по большей части касающийся пользовательского взаимодействия через графический интерфейс. Также будет добавлена возможность восстановлени файловой системы с помощью fsck. Будет добавлена возможность корректного завершения работы через графический интерфейс Phantom, так как сейчас для этого есть только заглушка. Для других компонентов системы это должно быть прозрачно и не потребует каких либо изменений. Помимо этого будут проведены работы по оптимизации, а компонент будет сконфигурирован для использования ключей с флешки.
  ```

- **en**

  ```
  Genode already includes a file\_vault component used to create secure storage. The Tresor library is used for encryption. We plan to adapt this component for encrypting snapshots. To do this, all unnecessary functionality—mostly related to user interaction via the graphical interface—will be removed. The ability to recover the file system using fsck will also be added. The ability to properly shut down via the Phantom GUI will be added, as currently there is only a placeholder for this. For other system components, this should be transparent and require no changes. In addition, optimization work will be carried out, and the component will be configured to use keys from a USB flash drive.
  ```



В Genode уже есть компонент file_vault, используемый для создания безопасного хранилища. Для шифрования используется библиотека Tresor. Планируется адаптироват этот компонент для шифрования снапшотов. Для этого будет убран весь нетребуемый функционал, по большей части касающийся пользовательского взаимодействия через графический интерфейс. Также будет добавлена возможность восстановлени файловой системы с помощью fsck. Будет добавлена возможность корректного завершения работы через графический интерфейс Phantom, так как сейчас для этого есть только заглушка. Для других компонентов системы это должно быть прозрачно и не потребует каких либо изменений. Помимо этого будут проведены работы по оптимизации, а компонент будет сконфигурирован для использования ключей с флешки.

