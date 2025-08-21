# Arkose-Long-Hold-Security-Bypass

This demonstrates how easy it is for a program to bypass the Arkose Labs/PX Long Hold Human Verification Challenge.

<img src="https://github.com/user-attachments/assets/af1bac21-637b-49bb-a23f-4bc83486d34c" width="300">

Demo code using [Camoufox](https://camoufox.com)
```
def handle_press_and_hold(page):
    """
    Check for and handle 'PRESS & HOLD' challenges in iframes (always in iframes)
        
    Args:
        page: Playwright page
    """
    try:
        # Check iframes for PRESS & HOLD elements (always in iframes)
        iframes = page.frames
        for frame in iframes:
            try:
                # Method 1: Look for "PRESS & HOLD" text
                press_hold_element = frame.get_by_text("PRESS & HOLD", exact=True)
                if press_hold_element.count() > 0:
                    print(f"{yellow_text}Found PRESS & HOLD challenge, handling...{reset}")
                    return handle_press_hold_element(frame, press_hold_element.first)
                
                # Method 2: Look for "Robot or human?" text
                robot_title = frame.get_by_text("Robot or human?", exact=False)
                if robot_title.count() > 0:
                    # Find the button with accessibility image or "Press" text
                    button_selectors = [
                        'div[role="button"]:has(svg)',  # Button with accessibility image
                        'div[role="button"]:has(p:has-text("Press"))',  # Button with "Press" text
                        'button:has(svg)',  # Button with accessibility image
                        'button:has(p:has-text("Press"))',  # Button with "Press" text
                        '[role="button"]',  # Any element with button role
                        'button',  # Any button element
                        'div:has-text("Press")',  # Any div containing "Press" text
                        'p:has-text("Press")'  # Any p containing "Press" text
                    ]
                    
                    for selector in button_selectors:
                        try:
                            button = frame.locator(selector).first
                            if button.count() > 0:
                                print(f"{yellow_text}Found robot challenge, handling...{reset}")
                                return handle_press_hold_element(frame, button)
                        except Exception as selector_error:
                            continue
                            
            except Exception as frame_error:
                continue
        
        return False
            
    except Exception as e:
        print(f"{red_text}Error handling PRESS & HOLD: {e}{reset}")
        return False
```


```
def handle_press_hold_element(page_or_frame, element):
    """
    Handle the actual press and hold action using bounding box method
        
        Args:
        page_or_frame: Page or frame object
        element: Element to press and hold
    """
    try:
        # Check if element is still attached to DOM and try to make it visible
        try:
            element.wait_for_element_state("visible", timeout=3000)
        except Exception as visibility_error:
            # Try to scroll element into view first
            try:
                element.scroll_into_view_if_needed()
                time.sleep(1.0)
            except Exception as scroll_error:
                pass
        
        # Final visibility check
        if not element.is_visible():
            return True  # Return True to indicate we should proceed to next step
        
        # Wait for element to be stable
        try:
            element.wait_for_element_state("stable", timeout=5000)
        except Exception as wait_error:
            pass
        
        # Get element bounding box with retry
        bounding_box = None
        max_retries = 3
        
        for attempt in range(max_retries):
            try:
                bounding_box = element.bounding_box()
                if bounding_box:
                    break
                else:
                    time.sleep(1.0)
            except Exception as bbox_error:
                if attempt < max_retries - 1:
                    time.sleep(1.0)
                else:
                    return False
        
        if not bounding_box:
            # Try alternative interaction method when bounding box fails
            try:
                # Move to element and click
                element.hover()
                time.sleep(0.5)
                
                # Simulate press and hold using mouse actions
                if hasattr(page_or_frame, 'mouse'):
                    # It's a page object
                    page_or_frame.mouse.down()
                    time.sleep(10)  # Hold for 10 seconds
                    page_or_frame.mouse.up()
                else:
                    # It's a frame object - use the page's mouse
                    page = page_or_frame.page
                    page.mouse.down()
                    time.sleep(10)  # Hold for 10 seconds
                    page.mouse.up()
                
                print(f"{green_text}✓ PRESS & HOLD completed{reset}")
                time.sleep(3.0)  # Increased wait time after successful alternative PRESS & HOLD
                return True
                
            except Exception as alt_error:
                return False
        
        # Validate bounding box values
        if (bounding_box['width'] <= 0 or bounding_box['height'] <= 0 or 
            bounding_box['x'] < 0 or bounding_box['y'] < 0):
            return False
        
        # Check if element is within viewport
        viewport_size = page_or_frame.viewport_size if hasattr(page_or_frame, 'viewport_size') else page_or_frame.page.viewport_size
        if (bounding_box['x'] + bounding_box['width'] > viewport_size['width'] or 
            bounding_box['y'] + bounding_box['height'] > viewport_size['height']):
            try:
                element.scroll_into_view_if_needed()
                time.sleep(1.0)
                # Get updated bounding box after scroll
                bounding_box = element.bounding_box()
                if not bounding_box:
                    return False
            except Exception as scroll_error:
                return False
        
        # Move to center of element
        x = bounding_box['x'] + bounding_box['width'] / 2
        y = bounding_box['y'] + bounding_box['height'] / 2
        
        # Apply offset as before
        x -= 150
        y += 150
        
        # Check if we're dealing with a frame or page
        if hasattr(page_or_frame, 'mouse'):
            # It's a page object
            page_or_frame.mouse.move(x, y)
            page_or_frame.mouse.down()
            time.sleep(12)  # Hold for 10 seconds
            page_or_frame.mouse.up()
        else:
            # It's a frame object - use the page's mouse
            page = page_or_frame.page
            page.mouse.move(x, y)
            page.mouse.down()
            time.sleep(10)  # Hold for 10 seconds
            page.mouse.up()
        
        print(f"{green_text}✓ PRESS & HOLD completed{reset}")
        time.sleep(3.0)  # Increased wait time after successful PRESS & HOLD
        return True
            
    except Exception as error:
        print(f"{red_text}PRESS & HOLD failed: {error}{reset}")
        return False
```
