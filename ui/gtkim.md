GDK: don't XFilterEvent key press/release

gtk_im_context --- gtk_im_context_simple
               \-- gtk_im_multicontext -- gtk_im_context_XXX in module form

gtk_im_context_simple: simple input method for Europe
gtk_im_multicontext: proxy for other im context
XIM im module: XFilterEvent

gtk_im_multicontext_{new,set_client_window,filter_keypress,focus_in,get_preedit_string,set_cursor_position}
