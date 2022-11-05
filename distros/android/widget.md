ArrayAdapter<T>
- ArrayAdapter<T> extends BaseAdapter and more
- BaseAdapter is an abtract class implementing SpinnerAdapter and more
- SpinnerAdapter is an interface extending Adapter

Spinner
- Spinner extends AbsSpinner and implements OnClickListener so that it
  can be AlertDialog's listener.
- AbsSpinner is an abstract class extending AdapterView<SpinnerAdapter>
- AdapterView<T> extends ViewGroup
- performClick is overridden.  Adapter set earlier is wrapped so that it can be
  passed to AlertDialog.Builder.  An alert dialog is shown.
- After an item is selected in alert dialog, spinner setSelection the item and
  dismisses the dialog.  If selection changes, requestLayout and invalidate.

Adapter
- is an interface for a set of data to be displayed
- has getView to return a View of an item
- ListAdapter for data expected to be displayed in a vertically scrolling list
- SpinnerAdapter for data expected to be displayed in a drop-down menu
- e.g. Spinner's drop-down menu invokes getDropDownView of a SpinnerAdapter;
       while its one-item menu invokes getView.

ArrayAdapter<T> revisited
- it stores a List of items
- it implements adapter interface by returning a TextView when
  being getView.

AdapterView<T>
- extends ViewGroup
- helps caches views returned by an adapter
- has the notion of position, used for current position or position changed

Dialog
- in ctor,
  mContext = new ContextThemeWrapper(ctx)
  mWindowManager = (WindowManager)context.getSystemService("window");
  mWindow = PolicyManager.makeNewWindow(mContext)
- the new (Phone)Window uses the given wm, which is assumed to be WindowManagerImpl.
- when a dialog is first shown, it calls onCreate and onStart
  the window's decor view is addView to the wm
- the usually path is followed, and a new ViewRoot is created and the decor
  becomes the view of ViewRoot.
- the view being setContentView is added as decor view's single child
- when dismiss, decor view is removeView from wm, and onStop is called
  wm's removeView die() the ViewRoot
