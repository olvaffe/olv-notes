Linux net bridge
================

## How Does It Work?

- `ip link set eth0 master br0` sends `RTM_SETLINK`
  - `rtnl_setlink` handles the request
  - `do_set_master` calls `upper_dev->netdev_ops->ndo_add_slave`
  - for a brige, this is `br_add_slave`
  - `br_add_slave` calls `netdev_rx_handler_register` with `br_handle_frame`
    on the slave device
- when the slave device receives an ethernet frame, `__netif_receive_skb_core`
  is called ultimately
  - `br_handle_frame` intercepts the handling of the skb and may return
    - `RX_HANDLER_CONSUMED`, which means the skb has been consumed by br
    - `RX_HANDLER_PASS`, which means the skb should be handled normally by the
      slave device
  - when the skb is consumed by br, it is handled by `br_handle_frame_finish`
    - if the skb is for the br, `br_pass_frame_up` points `skb->dev` to the br
      and calls `br_netif_receive_skb` to go through the regular processing
    - if the skb is for another machine, `br_forward` forwards it through the
      appropriate port
      - `__br_forward` pinits `skb->dev` to the slave dev of the port and
        calls `br_forward_finish`
      - `br_dev_queue_push_xmit` calls `dev_queue_xmit` to xmit the skb
