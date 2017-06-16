```pascal
unit FMX.WinForm;

interface

uses
  System.Classes,
  Winapi.Windows, Winapi.Messages,
  FMX.Forms;

type
  TWinForm = class(TForm)
  private
    FSavedWndProc: TFNWndProc;
    function GetWnd: HWND;
    function GetWndLong(AIndex: Integer): NativeInt;
    procedure SetWndLong(AIndex: Integer; const AValue: NativeInt);
  protected
    procedure InitMessageProxy; virtual;
    procedure WndProc(var Message: TMessage); virtual;
  public
    constructor Create(AOwner: TComponent); override;
    procedure DefaultHandler(var Message); override;
    property Wnd: HWND read GetWnd;
    property WndLong[Index: Integer]: NativeInt read GetWndLong write SetWndLong;
  end;

implementation

uses
  System.SysUtils,
  FMX.Platform.Win;

{ TWinForm }

constructor TWinForm.Create(AOwner: TComponent);
begin
  inherited;
  InitMessageProxy;
end;

function TWinForm.GetWnd: HWND;
begin
  Result := FormToHWND(Self);
end;

function TWinForm.GetWndLong(AIndex: Integer): NativeInt;
begin
  SetLastError(0);
  Result := GetWindowLong(Wnd, AIndex);
  if Result = 0 then
    CheckOSError(GetLastError);
end;

procedure TWinForm.SetWndLong(AIndex: Integer; const AValue: NativeInt);
begin
  SetLastError(0);
  if SetWindowLong(Wnd, AIndex, AValue) = 0 then
    CheckOSError(GetLastError);
end;

procedure TWinForm.InitMessageProxy;
begin
  FSavedWndProc := TFNWndProc(WndLong[GWL_WNDPROC]);
  WndLong[GWL_WNDPROC] := IntPtr(MakeObjectInstance(WndProc));
end;

procedure TWinForm.WndProc(var Message: TMessage);
begin
  Dispatch(Message);
end;

procedure TWinForm.DefaultHandler(var Message);
begin
  if FSavedWndProc <> nil then
    with TMessage(Message) do
      Result := CallWindowProc(FSavedWndProc, Wnd, Msg, WParam, LParam)
  else
    inherited DefaultHandler(Message);
end;

end.
```