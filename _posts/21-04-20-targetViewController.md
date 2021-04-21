---
layout: post
title: targetViewController
---

There is a senario where that we have view controller A, and A presents view controller B, then B presents C view controllers, and so on, this will form a view controller hierarchy. At a certain point, we may want to pass one piece of data from a far away child view controller, C for example, back to parent controller A. How do we do that?

I can think several solutions here:
1. Delegate
2. Notification
3. Reactive 
   
Apart from these, there is one more that I think could be used here.
Since iOS 8, Appple introduce a new method called `targetViewController(forAction:sender:)`,
Apple document describes this methos as:
> This method returns the current view controller if that view controller overrides the method indicated by the action parameter. If the current view controller does not override that method, UIKit walks up the view hierarchy and returns the first view controller that does override it. If no view controller handles the action, this method returns nil.

As it said this method basically just trace up the view controller hierarchy to find a proper receiver for the action. Let's see how to pass data back to parent by code.

First we have a controller A and its child view controller B, C, in C there is an action that will trigger the data passing back:

```
class AViewController: UIViewController, TrackingUp {
   
       @IBAction func next(_ sender: Any) {
        let vc = BViewController()
        addChild(vc)
        view.addSubview(vc.view)
        vc.didMove(toParent: self)
      ...
    }
}
class BViewController: UIViewController {

    @IBAction func next(_ sender: Any) {
        let vc = CViewController()
        self.show(vc, sender: nil)
        addChild(vc)
        view.addSubview(vc.view)
    ...
    }
}
class CViewController: UIViewController {
    
    @IBAction func clickeAction(_ sender: UIButton) {
    
        // do some thing here
    }
    
}
```
Then define an selector method:
```
@objc protocol TrackingUp: NSObjectProtocol {
    func didSelected(index: NSNumber)
}
```
**Note: the protocol has be anotated by `@objc`, so it is Objective-C visible.**
In `clickAction` function:
```
    
      @IBAction func clickeAction(_ sender: UIButton) {
    
        let selector = #selector(TrackingUp.didSelected(index:))
        guard let target = self.targetViewController(forAction: selector, sender: sender)  else { return } // 1
        target.perform(selector, with: 1)
    }
```
line 1 is the magic happen, by calling `targetViewController`, UIKit will trace up the hierarchy for us, so we just need to override `targetViewController` in the view controller A  you want to received that piece of data.
```
   override func targetViewController(forAction action: Selector, sender: Any?) -> UIViewController? {
        let selector = #selector(TrackingUp.didSelected(index:))
        if action == selector {
            return self
        }
        return super.targetViewController(forAction: action, sender: sender)
    }
```
Of cause we also need to impletement the protocol method in C controller:

```
extension AViewController: TrackingUp {
    func didSelected(index: NSNumber) {
        print("A: selected \(index)")
    }
}
```
### summary
> Pro
> 
- Use this method we can avoid to pass around delegate objects or introducing other reactive libraries.
- It also stick to build in UIKit which all iOS developers are familiar with.

> Con
> 
- You have to expose functions to Objective-C and use Objective-C runtime.
- In `performSelector:(SEL)aSelector withObject:(id)object` function argument object lost type information(e.g. an `id`), that is why the argument type defined in protocol `didSelected(index: NSNumber)` method is `NSNumber`.
- The child view controllers have to know what kind of the `selector` need to be checked, leaking unneeded details.
- If the final received view controller is container controller like `UINavigationViewController` or `TabViewController` then you may have to subclass your own container controller in order to override `targetViewController` function. 

***
Here is another good article regarding [ViewController Hierarchies](https://sandofsky.com/patterns/controller-hierarchies/)
[source code](https://gist.github.com/simeonlu/2d544ee1999e310836c6c7b5041f7393.js)
