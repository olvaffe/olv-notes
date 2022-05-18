Clipboard
=========

## X Window Selection

- <https://tronche.com/gui/x/icccm/sec-2.html>
- cut buffers
  - deprecated
  - each cut buffer is a property of the root window named `CUT_BUFFER1`, etc.
  - when user copies text in a client, the client updates the root window's
    cut buffer using `ChangeProperty`
  - when user pastes text in another client, the client gets the root window's
    cut buffer using `GetProperty`
- selections
  - there can be any number of selections
  - when user copies text in a client, the client requests to become the owner
    of a selection using `SetSelectionOwner`
  - when user pastes text in another client, the client gets the text from the
    owner through a sequence of operations
    - the client requests the owner to update a property of the client's
      window using `ConvertSelection`
    - the owner is notified by `SelectionRequest`
    - the owner updates the client's property using `ChangeProperty`
    - the owner sends `SelectionNotify` to the client
    - the client reads the property usign `GetProperty`
  - when the owner dies, the data is lost
- ICCCM defines three selections: `PRIMARY`, `SECONDARY`, and `CLIPBOARD`
  - they are all normal selections and behave the same
  - ICCCM just defines the conventions to use them
  - `PRIMARY` is the main selection for data exchange
  - `SECONDARY` is secondary but is not used nowadays
  - there may or may not be a special client that keeps becoming the owner of
    `CLIPBOARD`
    - when a client becomes the owner of `CLIPBOARD`, the special client
      retrieves the data and becomes owner again
    - when a client requests data from `CLIPBOARD`, the special client sends
      it the data
    - this keeps the data alive when the producer dies
