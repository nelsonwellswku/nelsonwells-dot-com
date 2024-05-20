+++
title = 'Update any index in CodeIgniter Cart class'
date = 2010-04-14T15:00:00-05:00
draft = false
tags = ['programming', 'php', 'codeigniter']
+++
If you use the Cart class that was introduced in CodeIgniter 1.7.2 very often, you'll notice that it has some shortcomings. In particular, the update() method will only update the quantity of the item in the cart; it will not update other indexes.

This is troublesome because sometimes you may want to update the price, for example, if the quantity goes above a certain threshold. Suppose you want to give a 10% discount on an item to anyone who orders 10 or more of that item. As it stands, you would have to remove the item from the cart, calculate the discount for the item, and re-add the item with a different price. I have also ran into the problem when I added a flag index to items in the cart to which I needed to do additional processing during checking depending on its status. I've found in both cases that the better solution is to extend the cart class. Here is how we can do it.

<!--more-->

```
class MY_Cart extends CI_Cart
{
    function __construct()
    {
        parent::CI_Cart();
    }

    function update_all($items = array())
    {
        // Was any cart data passed?
        if ( ! is_array($items) OR count($items) == 0)
        {
            return false;
        }

        // You can either update a single product using a one-dimensional array,
        // or multiple products using a multi-dimensional one.  The way we
        // determine the array type is by looking for a required array key named "rowid".
        // If it's not found we assume it's a multi-dimensional array
        if (isset($items['rowid']))
        {
            $this->_update_item($items);
        }
        else
        {
            foreach($items as $item)
            {
                $this->_update_item($item);
            }
        }

        $this->_save_cart();
    }

    /*
    * Function: _update_item
    * Param: Array with a rowid and information about the item to be updated
    *             such as qty, name, price, custom fields.
    */
    function _update_item($item)
    {
        foreach($item as $key => $value)
        {
            //don't allow them to change the rowid
            if($key == 'rowid')
            {
                continue;
            }

            //do some processing if qty is
            //updated since it has strict requirements
            if($key == "qty")
            {
                // Prep the quantity
                $item['qty'] = preg_replace('/([^0-9])/i', '', $item['qty']);

                // Is the quantity a number?
                if ( ! is_numeric($item['qty']))
                {
                    continue;
                }

                // Is the new quantity different than what is already saved in the cart?
                // If it's the same there's nothing to do
                if ($this->_cart_contents[$item['rowid']]['qty'] == $item['qty'])
                {
                    continue;
                }

                // Is the quantity zero?  If so we will remove the item from the cart.
                // If the quantity is greater than zero we are updating
                if ($item['qty'] == 0)
                {
                    unset($this->_cart_contents[$item['rowid']]);
                    continue;
                }
            }

            $this->_cart_contents[$item['rowid']][$key] = $value;
        }
    }
}
```

Copy and paste the previous code into a file named MY_Cart.php and place it in the library folder in your application folder. To use the extension to the cart class is as easy as using the normal update method from the cart class. You can pass an array to update one item, or you may pass a multi-dimensional array to update multiple items.

    $data = array(
        'rowid' => 'rowid_from_post',
        'qty' => 10,
        'name' => 'hello'
    );

    $this->cart->update_all($data);

or

    $data = array(
        array(
            'rowid' => 'rowid_from_post',
            'name' => 'world'
        ),
        array(
            'rowid' => 'rowid_from_post',
            'price' => 25.50
        )
    );

    $this->cart->update_all($data);

## Limitations

There is one thing I forgot to mention. When updating an item or items, this extension will not work with the 'options' index or any user-defined index that is an array. If you need to edit the options array index in-cart, then you'll have to think of another solution.

I plan on adding features to this cart extension, and if it gets robust enough, I will re-write the extension into the cart itself.

Happy coding!
