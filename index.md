## Обработка сообщений Windows в приложениях FireMonkey

### Введение

Сообщения &mdash; это основная форма взаимодействия между системой и&nbsp;приложениями в&nbsp;Windows. С&nbsp;помощью сообщений операционная система уведомляет приложения о&nbsp;различных событиях, источниками которых могут быть как сама система, так и&nbsp;другие приложения. Адресатами сообщений являются окна приложения и&nbsp;их&nbsp;контролы (элементы интерфейса), которые обрабатывают их&nbsp;с&nbsp;помощью специальной [оконной процедуры](https://msdn.microsoft.com/en-us/library/windows/desktop/ms632593(v=vs.85).aspx).

В&nbsp;VCL-приложениях Delphi оконная процедура `WndProc` присутствует во&nbsp;всех наследниках класса `TControl`&nbsp;и, либо обрабатывает входящее сообщение непосредственно внутри себя, либо [передаёт его обработчику](http://docwiki.embarcadero.com/Libraries/Berlin/en/System.TObject.Dispatch) с&nbsp;помощью метода `Dispatch`. Таким образом у&nbsp;разработчика VCL-приложения есть сразу несколько способов обработки сообщений:
* Перегрузить метод `WndProc`
* Добавить обработчик событий на&nbsp;определённое сообщение:
  ```pascal
  procedure WMPaint(var Message: TWMPaint); message WM_PAINT;
  ```
* Перегрузить метод `DefaultHandler`, который будет вызван в&nbsp;случае, если `Dispatch` не&nbsp;найдёт обработчик для полученного сообщения

В&nbsp;FireMonkey-приложении за&nbsp;взаимодействие с&nbsp;операционной системой отвечает специальный объект &mdash; &laquo;платформа&raquo;, чей класс зависит от&nbsp;выбранной платформы. &laquo;Платформа&raquo; и&nbsp;её&nbsp;класс объявлены в&nbsp;`implementation`-секции своего модуля и&nbsp;доступны только через [набор интерфейсов служб](http://docwiki.embarcadero.com/RADStudio/Tokyo/en/FireMonkey_Platform_Services), а&nbsp;также посредством [системы сообщений RTL](http://docwiki.embarcadero.com/RADStudio/Tokyo/en/Using_the_RTL_Cross-Platform_Messaging_Solution).

При создании FMX-приложения для Windows &laquo;платформой&raquo; будет объект класса `TPlatformWin` объявленный в&nbsp;`FMX.Platform.Win.pas`. Внутри `TPlatformWin` находится собственная оконная процедура `WndProc`, обрабатывающая сообщения Windows, однако в&nbsp;отличие от&nbsp;VCL-приложения, полученное сообщение никогда не&nbsp;будет передано форме через `Dispatch`. Невозможность наследования класса окончательно лишает нас возможность обработать сообщение Windows обычными методами

У&nbsp;FMX-формы есть свой дескриптор окна типа `TWindowHandle`, но&nbsp;он&nbsp;несовместим с&nbsp;типом `HWND`. Однако в&nbsp;уже упомянутом выше модуле `FMX.Platform.Win.pas` есть публичная процедура `WindowHandleToPlatform`, которая преобразует `TWindowHandle` в&nbsp;`TWinWindowHandle` содержащий, помимо всего прочего, дескриптор окна Windows. Также в&nbsp;последних версиях Delphi там появился упрощённое получение дескриптора с&nbsp;помощью функции `FormToHWND`.

### Перехват

Итак, необходимо сделать свой обработчик сообщений Windows имея в&nbsp;своём распоряжении только дескриптор окна. Изучив варианты, я&nbsp;остановился следующем:
* Создаём новый класс-наследник от&nbsp;`TForm`, добавляем к&nbsp;нему метод, который будет нашей новой оконной процедурой:
```pascal
procedure TWinForm.WndProc(var Message: TMessage);
begin
  Dispatch(Message);
end;
```
* В&nbsp;конструкторе формы с&nbsp;помощью `GetWindowLong` получаем адрес текущей оконной процедуры и&nbsp;сохраняем его. Затем, с&nbsp;помощью `SetWindowLong` задаём в&nbsp;качестве оконной процедуры наш недавно созданный метод `WndProc`.
```pascal
TWinForm = class(TForm)
private
  FSavedWndProc: TFNWndProc;
```
```pascal
constructor TWinForm.Create(AOwner: TComponent);
begin
  inherited;
  FSavedWndProc := TFNWndProc(GetWindowLong(
    FormToHWND(Self), 
    GWL_WNDPROC
  ));
  SetWindowLong(
    FormToHWND(Self), 
    GWL_WNDPROC, 
    IntPtr(MakeObjectInstance(WndProc)) 
  );
end;
```
* Перегружаем метод `DefaultHandle` в&nbsp;теле которого вызываем ранее сохранённую оконную процедуру. Таким образом, если после `Dispatch` перехваченное сообщение не&nbsp;будет обработано, то&nbsp;оно автоматически будет передано в&nbsp;исходный обработчик внутри `TPlatformWin`.
```pascal
procedure TWinForm.DefaultHandler(var Message);
begin
  if FSavedWndProc <> nil then
    with TMessage(Message) do
  Result := CallWindowProc(FSavedWndProc, Wnd, Msg, WParam, LParam);
end;
```
