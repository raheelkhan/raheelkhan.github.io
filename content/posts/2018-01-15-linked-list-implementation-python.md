---
title: Linked list implementation in Python
date: 2018-01-15
categories:
---

```python
class Node(object):
    """
    A Node is a single element in a linked list datastructure. It consists of two parts
        1 - Data (Can be of any type)
        2 - A reference to the next Node
    If the next_node pointers happens to be null (None) it indicates that this is the
    last element (Node) of the linked list
        :param object:
    """
    def __init__(self, data, next_node):
        self.data = data
        self.next_node = next_node


class LinkedList(object):
    """
    A LinkedList class acts as a containers of all nodes.
    We add an element into linked list by calling the add method with the data
    In order to perform operations on a linked list we need a starting point i.e
    the first node. This first node is always bind to the head of link. If
    the list is empty the head variable will be None
        :param object:
    """

    def __init__(self):
        """
            :param self:
        """
        self.head = None
        self.size = 0

    def add(self, data):
        """
        To add an element in a linked list we first create a node object with the data
        and setting the head of linked list as pointer to the next_node of the node object
        After creating a new node we set the head of linked list pointing to the last created node
        ******************************
        | Node C -> Node B -> Node A | The last added not will always be on head
        ******************************
        we also maintain the size of the linked list by incrementing it by 1 for every add operation
            :param self
            :param data
        """
        node = Node(data, self.head)
        self.head = node
        self.size += 1

    def find(self, data):
        """
        To find an element in a linked list we start with the head of linked list.
        We set a pointer variable current_node to the head and set a found flag to False
        Then we loop by checking the current_node variable truthness. For every iteration we check
        the current node data property to the value provided in the argument. If the value is matched
        we set the found flag to True and break the loop. Otherwise we assign the next_node property to
        current_node pointer to keep looping unless the next_node happens to be equal to None.
        If the loop finishes without breaking it means that we have reached the last node without finding
        the value thus return False
            :param self
            :param data
        """
        current_node = self.head
        found = False
        while current_node:
            if current_node.data == data:
                found = True
                break
            current_node = current_node.next_node
        return found

    def remove(self, data):
        """
        To remove element from linked list we have to first find the node
        We follow the same approach as in the find method except instead of returning True or False
        We shift the pointer of previous_node to point to the next_node of current_node.
        This will not actually delete the object but take it out of the chain.
            :param self:
            :param data:
        """
        current_node = self.head
        previous_node = None
        while current_node:
            if current_node.data == data:
                if previous_node is None:
                    self.head = current_node
                else:
                    previous_node.next_node = current_node.next_node
                self.size -= 1
                return True
            previous_node = current_node
            current_node = current_node.next_node
        return False


if __name__ == '__main__':
    linked_list = LinkedList()
    """
    Lets add few elements into our linked list
    """
    linked_list.add(9)
    linked_list.add(2)
    linked_list.add(7)
    linked_list.add(8)

    """
    Find element with value 2
    """
    print linked_list.find(2)

    """
    Remove element with value 8
    """
    print linked_list.remove(8)

    """
    Check the size of list after removing one element
    """
    print linked_list.size
```
