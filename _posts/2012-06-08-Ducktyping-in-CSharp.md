---
layout: post
title: Wenn es watschelt und quakt&#58; Ducktyping in C#
tags: .net c# duck-typing dynamic
excerpt_separator: <!--more-->
---

> *"When I see a bird that walks like a duck and swims like a duck and quacks like a duck, I call that bird a duck."*  
> — James Whitcomb Riley

Oder auf gut Deutsch: "Wenn ich einen Vogel sehe, der wie eine Ente watschelt und wie eine Ente quakt, dann ist dieser Vogel für mich eine Ente." Das selbe Prinzip das James Whitcomb Riley auf die Enten angewandt hat, kann man auch in der Software-Entwicklung anwenden. Zwei Fragen stellen sich dabei zwangsläufig. Ist das wirklich eine gute Idee? Und wenn ja, wie geht das?<!--more-->

Kritiker des Ducktypings werden sagen, wenn ein Objekt einer Klasse in einen anderen Typ umgewandelt werden soll, muss der neue Typ eben vom alten Typ abgeleitet sein. Diese Aussage ist absolut korrekt und wo das möglich ist, sollte das auch so gehandhabt werden. Allerdings gibt es Szenarien in denen genau das nicht der Fall ist, und in diesen Fällen kann Ducktyping Abhilfe schaffen. Nehmen wir an, wir hätten ein Interface vom Typ `IEnte` und eine Klasse vom Typ `Vogel`:

````csharp
public interface IEnte 
{
    void Watscheln();
    void Quaken();
}
````
    
````csharp
public class Vogel 
{
    public void Watscheln() { ... }
    public void Quaken() { ... }
}
````
    
Die Klasse `Vogel` hat zwar die Methoden `Watscheln()` und `Quaken()`, leitet aber (aus welchen Gründen auch immer) nicht vom Interface `IEnte` ab. Hier braucht man also einen Wrapper, der ein Objekt vom Typ `Vogel` in einer Klasse verpackt, die `IEnte` implementiert. Das hört sich leicht an, hat aber seine Tücken - und war vor .NET 4 unmöglich. Denn ein Objekt vom `Vogel` kann nicht auf `IEnte` gecastet werden, das würde einen Compilerfehler verursachen. An dieser Stelle kommt das keyword `dynamic` ins Spiel. Die [MSDN-Dokumentation][1] sagt dazu:

> "Der `dynamic`-Typ ermöglicht die Vorgänge, in denen die Überprüfung des Kompilierzeittyps umgangen wird. Stattdessen werden diese Vorgänge zur Laufzeit aufgelöst."

Die Implementierung einer Wrapper-Klasse könnte also so aussehen:

````csharp 
public class SiehtAusWieEineEnte : IEnte
{
    private readonly dynamic _ente;

    public SiehtAusWieEineEnte(dynamic ente)
    {
        this._ente = ente;
    }

    public void Quaken()
    {
        this._ente.Quaken();
    }

    public void Watscheln()
    {
        this._ente.Watscheln();
    }
}
````

Die Überprüfung ob `this._ente` überhaupt über die Methoden `Watscheln()` und `Quaken()` verfügt erfolgt nicht zur Compilezeit, sondern zur Laufzeit. Und so kann man mit folgenden Zeilen ein Objekt des Typs `Vogel` in einem Objekt, das das Interface `IEnte` implementiert, verpacken:

````csharp
var vogel = new Vogel();
var ente = new SiehtAusWieEineEnte(vogel);
ente.Watscheln();
ente.Quaken();
```` 

 [1]: http://msdn.microsoft.com/de-de/library/dd264741.aspx