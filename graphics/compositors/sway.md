Sway
====

## running sway from ssh session

- `WLR_LIBINPUT_NO_DEVICES=1 WLR_SESSION=noop sway -d`


## `swaymsg -t get_tree`

- when there is an Alacritty window on the screen
  - node
    - `type`: `root`
    - `name`: `root`
    - node
      - `type`: `output`
      - `name`: `__i3`
      - node
        - `type`: workspace
        - `name`: `__i3_scratch`
    - node
      - `type`: `output`
      - `name`: `eDP-1`
      - node
        - `type`: `workspace`
        - `name`: `1`
        - node
          - `type`: `con`
          - `name`: current window title
          - `pid`: process pid
          - `app_id`: `Alacritty`
          - `shell`: `xdg_shell`
