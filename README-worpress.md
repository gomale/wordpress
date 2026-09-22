<a id="top"></a>

# WordPress Setup

## Table of Contents

- [Install Themes](#install-themes)
- [Build your site with Elementor](#build-your-site-with-elementor)
- [Set up WPForms](#set-up-wpforms)
- [Troubleshooting Guid](#troubleshooting-guide)
- [Reference](#reference)

### Install Themes

1. Click on Appearance --> Themes --> Add Theme --> Search Bar: Astra (click on Activate)

   Astra will be used for the Header and Footer

2. Click on **Start Buildig Now (Let's Get Started)**

3. Skip on **Get the Best Start with Astra**

4. Skip on **Select Your Features**

5. Click on **Build with Templates** on *Build Your Site in Minutes Using Pre-Built Templates*

6. Click on **Elementor** on *Select Page Builder*

7. Click on your preferred *Template* (e.g.: **Visual Artist Portfolio**)

8. Choose your desired **Font Pair** and **Color Palette*, then click on **Continue**

9. Leave the basic ones on **Select features**, ensure that **Elementor** is included in the plugins.

10. See the Troubleshooting Guide if you encounter these issues:

        Insufficient Memory Limit
        Low Max Execution Time

11. Click on **Submit & Build My Website**

12. Wait for the process to complete (**We are building your website…**)

13. Click on **View Your Website**

## Build your site with Elementor

Elementor works in Sections, Containers, and Elements.

1. Click on **Publish** to save your work.

2. On the Elementor menu, click on **Exit to WordPress**

3. Click on the **WordPress** symbol, then **Leave**

## Set up WPForms

1. Go to *Plugins --> Add Plugin*

2. Search for **wpforms**, then click on **Install Now** (Activate)

3. From the **WordPress** menu, click on *WPForms --> Add New Form*

4. Name Your Form: **New Contact Form** --> **Simple Contact Form** (Use Template)

5. Start editing, just like *Elementor*

6. Once everything is set, click on **Save** button and exit by clicking on the **X** icon

7. Refer to the [video tutorial](#reference) on the detailed steps.


[↑ Back to Top](#top)

## Troubleshooting Guide

### Insufficient Memory Limit and Low Max Execution Time

The template importer reports that PHP’s memory and execution time limits are too low. Try 512 MB memory and 300 seconds execution time; *the importer’s exact requirements depend on the plugin.*

For the Ubuntu 24.04 + Apache + PHP 8.3 setup above, run these commands on your Azure VM.

1. Increase PHP limits

   Open Apache’s PHP configuration:

   ```bash
   sudo nano /etc/php/8.3/apache2/php.ini
   ```

   Find and update the existing settings:

   ```ini
   memory_limit = 512M
   max_execution_time = 300
   ```

   Save with Ctrl+O, press Enter, then exit with Ctrl+X.

   These settings control memory available to each PHP script and its execution time limit.

2. Update WordPress memory limits

   ```bash
   sudo nano /var/www/wordpress/wp-config.php
   ```

   Add these lines before the /* That's all, stop editing! ... */ comment. If they already exist, update them rather than adding duplicates:

   ```php
   define( 'WP_MEMORY_LIMIT', '256M' );
   define( 'WP_MAX_MEMORY_LIMIT', '512M' );
   ```

   WP_MAX_MEMORY_LIMIT controls the memory WordPress requests for administration tasks, including many imports.

3. Apply the changes

   ```bash
   sudo apache2ctl configtest
   ```

   If it reports **Syntax OK**, then restart Apache

   ```bash
   sudo systemctl restart apache2
   ```

4. Verify in WordPress

   Go to Dashboard → Tools → Site Health → Info → Server and check:

   | Setting | Expected value |
   |---|---|
   | PHP memory limit | `512M`, or a separate administration limit of `512M` |
   | PHP time limit | `300` |

   Refresh the template importer and retry.

   If the values do not change, check the PHP SAPI and PHP version in that same screen. If SAPI is fpm-fcgi, your site uses PHP-FPM: edit /etc/php/8.3/fpm/php.ini instead and restart php8.3-fpm. 
   
   Replace 8.3 with the version shown. Terminal PHP settings can differ from those used by the website.

[↑ Back to Top](#top)

## Reference

[WordPress Tutorial for Beginners (2026)](https://www.youtube.com/watch?v=KCTVfi_LNdU)

[↑ Back to Top](#top)