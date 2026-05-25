# Исследование и адаптация компонентов персистентной Phantom ОС на Genode

# Research and adaptation of components from the persistent OS Phantom for Genode





**EUMEL** - персистентная ОСь 1979 года разработки

Нужно сказать про отличия от гибернации

The advantage of orthogonal persistence environments is simpler and less error-prone programs. (https://en.wikipedia.org/wiki/Persistence_(computer_science)#Orthogonal_or_transparent_persistence)

https://en.wikipedia.org/wiki/Capability-based_security





## Abstract

- **ru**

  ```
  Эволюция операционных систем на протяжении десятилетий определялась концептуальной моделью, сформировавшейся в середине XX века с появлением UNIX. Эта модель, основанная на триаде «программа-процесс-файл», доказала свою жизнеспособность, однако сегодня она всё чаще демонстрирует фундаментальные ограничения \cite{OS}. Попытки обойти эти ограничения привели в том числе к появлению концепции операционных систем с персистентной памятью еще в конце 80-х годов XX века /cite{может какую ссылку вставить}. Несмотря на достоинства  такой парадигмы \cite{найти ч-н} исследования столкнулись с аппаратными ограничениями (?). С тех пор производительность компьютеров сильно выросла, что дало толчок новым работам в этом направлении. Яркой иллюстрацией этого является PhantomOS. Однако написание и внедрение полностью новой операционной системы весьма длительно, трудоемко и влечет за собой большое количество ошибок. Смягчить эти последствия помогает использование уже отработанных механизмов, чему способствует портирование на Genode /cite{ссылка на Genode}. В данной работе мы рассматриваем процесс адаптации компонентов PhantomOS на Genode. Мы описываем трудности, с которыми приходится сталкиваться, недостатки ОС и рассказываем о решениях, которые помогают избавиться от этих проблем или смягчить их.
  ```

  

- **en**

  ```
  The evolution of operating systems over the decades has been shaped by a conceptual model that emerged in the mid-20th century with the advent of UNIX. This model, based on the “program-process-file” triad, has proven its viability; however, today it increasingly reveals fundamental limitations \cite{OS}. Attempts to circumvent these limitations led, among other things, to the emergence of the concept of operating systems with persistent memory as early as the late 1980s /cite{maybe insert a link here}. Despite the merits of such a paradigm \cite{find a source}, research ran into hardware limitations (?). Since then, computer performance has increased significantly, which has spurred new work in this direction. PhantomOS is a striking illustration of this. However, writing and implementing a completely new operating system is a lengthy, labor-intensive process that is prone to numerous errors. The use of proven mechanisms helps mitigate these consequences, facilitated by porting to Genode /cite{link to Genode}. In this paper, we examine the process of adapting PhantomOS components to Genode. We describe the challenges encountered, the OS’s shortcomings, and discuss solutions that help eliminate or mitigate these issues.
  ```

## Introduction

