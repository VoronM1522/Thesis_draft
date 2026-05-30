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
  
  ```

- **en**

  ```
  
  ```

1. В чем заключается философия системы? Что про нее можно сказать (какие особенности можно выделить)?
   PhantomOS - это операционная система с открытым исходным кодом. Автором идеи является Дмитрий Завалишин. Он же внес основной вклад в разработку системы. PhantomOS предоставляет ортогонально персистентное окружение прикладным программам внутри Phantom Virtual Machine (PVM), внутри которой исполняется код на языке программирования Phantom. Однако стоит отметить, что имеется и подсистема совместимости POSIX.
2. Какие компоненты и как позволяют реализовать концепт ортогональной персистентности (сборщик мусора, механизм снапшотов и т. д.)
3. Как работает процесс создания снимка на оригинальной версии ОС?
4. Какова стоимость персистентности?

### PhantomOS на Genode

- **ru**

  ```
  
  ```

- **en**

  ```
  
  ```

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
  
  ```

- **en**

  ```
  
  ```

1. Зачем нужна защита данных в покое?
2. В чем отличие данных снапшотов и данных на диске?
3. Какие есть способы защиты данных в покое?
   1. Какие уровни шифрования бывают?
   2. В чем приемущества и недостатки шифрования на разных уровнях?
   3. Чем мы жертвуем? как это сказывается на производительности? (Аппаратное смягчение последствий)
   4. Есть ли другие способы защиты данных в покое?
4. Какой подход обычно применяется (или несколько)?
5. Какой способ предпочтителен для нас и почему? (возможно, это уже другая глава)

