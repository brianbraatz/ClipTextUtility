
~~~c
#include <windows.h>
#include <shellapi.h>
#include <string.h>

#define WM_TRAYICON (WM_USER + 1)
#define ID_TRAY_EXIT 1001

HINSTANCE hInst;
NOTIFYICONDATA nid;

void RemoveTrayIcon() {
    Shell_NotifyIcon(NIM_DELETE, &nid);
}

void ClearAndConvertClipboard(HWND hwnd) {
    if (OpenClipboard(hwnd)) {
        HANDLE hClipboardData = GetClipboardData(CF_TEXT);
        if (hClipboardData) {
            char *clipText = (char *)GlobalLock(hClipboardData);
            if (clipText) {
                // Create a new global memory block for the cleaned-up text
                size_t len = strlen(clipText);
                HGLOBAL hNewClipboardData = GlobalAlloc(GMEM_MOVEABLE, len + 1);
                if (hNewClipboardData) {
                    char *newClipText = (char *)GlobalLock(hNewClipboardData);
                    if (newClipText) {
                        strcpy(newClipText, clipText);
                        GlobalUnlock(hNewClipboardData);

                        EmptyClipboard(); // Clear the clipboard

                        // Set the new clipboard data
                        SetClipboardData(CF_TEXT, hNewClipboardData);
                    }
                }
            }
            GlobalUnlock(hClipboardData);
        }
        CloseClipboard();
    }
}

LRESULT CALLBACK WindowProc(HWND hwnd, UINT uMsg, WPARAM wParam, LPARAM lParam) {
    switch (uMsg) {
        case WM_TRAYICON:
            if (lParam == WM_RBUTTONUP) {
                // Show a context menu
                HMENU hMenu = CreatePopupMenu();
                AppendMenu(hMenu, MF_STRING, ID_TRAY_EXIT, "Exit");
                
                POINT cursor;
                GetCursorPos(&cursor);
                SetForegroundWindow(hwnd);
                int cmd = TrackPopupMenu(hMenu, TPM_RETURNCMD | TPM_NONOTIFY, cursor.x, cursor.y, 0, hwnd, NULL);
                if (cmd == ID_TRAY_EXIT) {
                    RemoveTrayIcon();
                    PostQuitMessage(0);
                }
                DestroyMenu(hMenu);
            } else if (lParam == WM_LBUTTONDBLCLK) {
                ClearAndConvertClipboard(hwnd);
            }
            break;

        case WM_DESTROY:
            RemoveTrayIcon();
            PostQuitMessage(0);
            break;

        default:
            return DefWindowProc(hwnd, uMsg, wParam, lParam);
    }
    return 0;
}

int APIENTRY WinMain(HINSTANCE hInstance, HINSTANCE hPrevInstance, LPSTR lpCmdLine, int nCmdShow) {
    hInst = hInstance;

    // Register a window class
    WNDCLASS wc = {0};
    wc.lpfnWndProc = WindowProc;
    wc.hInstance = hInstance;
    wc.lpszClassName = "ClipTextClass";
    RegisterClass(&wc);

    // Create an invisible window
    HWND hwnd = CreateWindow("ClipTextClass", "ClipText", 0, 0, 0, 0, 0, NULL, NULL, hInstance, NULL);

    // Add the icon to the system tray
    nid.cbSize = sizeof(NOTIFYICONDATA);
    nid.hWnd = hwnd;
    nid.uID = 1;
    nid.uFlags = NIF_ICON | NIF_MESSAGE | NIF_TIP;
    nid.uCallbackMessage = WM_TRAYICON;
    nid.hIcon = LoadIcon(NULL, IDI_APPLICATION); // Default application icon
    strcpy(nid.szTip, "Clip Text");

    Shell_NotifyIcon(NIM_ADD, &nid);

    // Message loop
    MSG msg;
    while (GetMessage(&msg, NULL, 0, 0)) {
        TranslateMessage(&msg);
        DispatchMessage(&msg);
    }

    return (int)msg.wParam;
}

~~~