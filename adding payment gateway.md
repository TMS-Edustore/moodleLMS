Go to Site Administration > Appearance > Additional HTML.
Paste this script in the "Within HEAD" section

<script>
    document.addEventListener("DOMContentLoaded", function () {
        let url = new URL(window.location.href);
        if (url.pathname.includes('/enrol/index.php')) {
            // Redirect to a custom pre-registration page before enrollment
            window.location.href = "http://localhost:8084";
        }
    });
</script>


To skip the redirect for users who have already paid, you need to check if they are already enrolled in the course. Since users are only enrolled after payment, this acts as a check for payment status.

Steps to Modify the JavaScript Redirect:
We need to check if the user is already enrolled in the course before redirecting. Since Moodle stores enrollment status in the course page, we can use an AJAX request to check this.

<script>
    document.addEventListener("DOMContentLoaded", function () {
        let url = new URL(window.location.href);
        let params = new URLSearchParams(url.search);
        let courseId = params.get("id"); // Get course ID from URL

        // Define an array of course IDs that require redirection
        let restrictedCourses = ["2", "5", "10"]; // Change to your actual course IDs

        // Function to check if the user is already enrolled
        function checkEnrollment(courseId, callback) {
            fetch(`/course/view.php?id=${courseId}`)
                .then(response => response.text())
                .then(html => {
                    // If the course page contains "Enrolled courses", the user is already enrolled
                    if (html.includes('id="page-course-view"')) {
                        callback(true); // User is enrolled
                    } else {
                        callback(false); // User is not enrolled
                    }
                });
        }

        // If the course is restricted, check enrollment status
        if (url.pathname.includes('/enrol/index.php') && restrictedCourses.includes(courseId)) {
            checkEnrollment(courseId, function (isEnrolled) {
                if (!isEnrolled) {
                    // Redirect only if the user is NOT enrolled
                    window.location.href = "https://yourwebsite.com/pre-registration-page";
                }
            });
        }
    });
</script>


Alternative: Checking Enrollment Using PHP (More Reliable)
Instead of checking enrollment using JavaScript, you can do it using PHP inside Moodle’s core files.

Modify the enrol/index.php file:
Location: moodle/course/enrol.php

Add this code before showing enrollment options:
Find this line (near the top of the file):
require_once(__DIR__ . '/../config.php');
Add the following check below it to redirect users who are not enrolled:

require_login();
global $USER, $DB, $COURSE;

// Get course ID from the URL
$courseid = required_param('id', PARAM_INT);
$context = context_course::instance($courseid);

// Check if the user is already enrolled
if (!is_enrolled($context, $USER->id)) {
    // Redirect to the pre-registration page if not enrolled
    redirect(new moodle_url('/your-custom-pre-registration-page.php'));
}


Alternative: Keep the Page and Just Show the Popup
If you don't want to stop execution (die()), you can still load the page but trigger the popup automatically.

Modify index.php:

require('../config.php');
require_login();
global $USER, $DB, $COURSE;

$courseid = required_param('id', PARAM_INT);
$context = context_course::instance($courseid);

$show_payment_popup = false; // Default: don't show the popup

if (!is_enrolled($context, $USER->id)) {
    $show_payment_popup = true; // User is not enrolled → Show payment popup
}

require_once("$CFG->libdir/formslib.php");
?>

<!-- Include Paystack Payment UI -->
<?php include_once(__DIR__ . '/paystack_payment.php'); ?>

<script>
document.addEventListener("DOMContentLoaded", function() {
    let showPopup = <?php echo json_encode($show_payment_popup); ?>;
    if (showPopup) {
        document.getElementById("payNow").click(); // Auto-trigger the payment popup
    }
});
</script>


