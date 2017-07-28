---
layout: post
title:  Reflection ohne Reflection: Properties lesen und setzen per Databinding 
---

Nehmen wir an, wir entwickeln ein UserControl, dass die DependencyProperty `TextPath` vom Typ `string` bereitstellt. Diese Property kann der Anwender im XAML-Code setzen um das Steuerelement zu veranlassen um den Wert einer bestimmten Property seines DataContexts auszulesen oder zu setzen. Das riecht danach, das Problem mit handelsüblicher Reflection zu lösen:

    var propertyInfo = this.DataContext
        .GetType()
        .GetProperty(this.TextPath);
    var result = propertyInfo.GetValue(this.DataContext, null);
    

Das klappte auch hervorragend. Bis einer der Anwender der Komponente auf die (absolut nachvollziehbare) Idee kam, den Namen der auszulesenden Property under Verwendung der [Property Path Syntax][1] anzugeben:

    <MyControl TextPath="Address.City" />
    

Und dann stellt man plötzlich fest, dass man mit Reflection in diesem Fall nicht sehr weit kommt. Theoretisch wäre eine Lösung zwar möglich, aber nur, wenn man bereit ist einen Parser für die Property Path Syntax zu schreiben und noch mehr Reflection in den Ring zu schicken.

Es gibt allerdings eine Komponente in Silverlight und im .NET-Framework die diese Funktionalität schon fix und fertig liefert: Die Klasse `System.Windows.Data.Binding`. Das Prinzip ist einfach: Man bindet die Quelle (in diesem Fall den DataContext des Steuerelements) unter Verwendung des angegebenen Pfades an eine Property einer Helferklasse und lasse das Binding seine Arbeit tun. Wie das konkret vonstatten geht?<!--more-->

Erste Zutat ist die Helferklasse. Dies muss von `DependencyObject` ableiten und eine `DependencyProperty` bereitstellen an die man später binden kann:

    public class BindingEvaluator : DependencyObject
    {
        public static readonly DependencyProperty TargetProperty = 
            DependencyProperty.Register("Target", 
                                        typeof(object), 
                                        typeof(BindingEvaluator), 
                                        new PropertyMetadata(default(object)));
    
        public object Target
        {
            get { return this.GetValue(TargetProperty); }
            set { this.SetValue(TargetProperty, value); }
        }
    }
    

Für die eigentliche Auswertung erstellt man [Extension-Methoden][2] die es ermöglichen, den Wert einer Property eines Objektes sowohl auszulesen als auch zu setzen. Dazu wird ein Binding erstellt, das als Quelle das Zielobjekt erhält und als Pfad den angegeben Property Path. Anschließend kann über die `Target`-Property der `BindingEvaluator`-Klasse der Wert gelesen oder geschrieben werden:

    public static class ObjectExtensions
    {
        public static object GetPropertyValue(this object source, 
                                              string propertyPath)
        {
            var binding = new Binding { Source = source, 
                                        Path = new PropertyPath(propertyPath)};
    
            var evaluator = new BindingEvaluator();
            BindingOperations.SetBinding(evaluator, 
                                         BindingEvaluator.TargetProperty,
                                         binding);
    
            return evaluator.Target;
        }
    
        public static void SetPropertyValue(this object source, 
                                            string propertyPath,
                                            object value)
        {
            var binding = new Binding { Source = source,
                                        Path = new PropertyPath(propertyPath),
                                        Mode = BindingMode.TwoWay };
    
            var evaluator = new BindingEvaluator();
            BindingOperations.SetBinding(evaluator, 
                                         BindingEvaluator.TargetProperty,
                                         binding);
    
            evaluator.Target = value;
        }
    }
    

Dabei ist zu beachten, dass in der Methode `SetPropertyValue` das Binding als TwoWay-Binding konfiguriert wird, da der Wert ansonsten nicht auf das Zielobjekt zurückgeschrieben wird.

Aufgerufen werden die Methoden dann direkt auf dem Zielobjekt:

    var city = this.DataContext.GetPropertyValue("Address.City");
    this.DataContext.SetPropertyValue("Address.City", "Hannover");
    

Keine Lösung kommt ohne Nachteile aus. Es ist unumstritten, dass Reflection teuer ist. Aber: Die hier vorgestellte Variante ist noch teurer. Liest man einen einfach verschachtelten Property Path (also z.B. "Address.City") eine Million mal mit Reflection aus dauert das insgesamt knapp 600 Millisekunden. Verwendet man zum Lesen der Daten stattdessen den oben vorgestellten Ansatz dauert der Vorgang gute 26 Sekunden!

**Fazit:** Muss man nur ab und an den Wert einer Property unter Verwendung eines Property Path auslesen ist der hier vorgestellte Weg eine gute Möglichkeit. Zur Massenverarbeitung ist er aber definitiv ungeeignet.

 [1]: http://msdn.microsoft.com/en-us/library/cc645024%28v=vs.95%29.aspx
 [2]: http://msdn.microsoft.com/en-us//library/bb383977.aspx